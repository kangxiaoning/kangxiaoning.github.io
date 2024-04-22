# 更新A记录后容器仍解析到旧IP

<show-structure depth="3"/>

## 1. 问题描述

应用A部署在Kubernetes集群中，Redis部署在集群外，应用A需要访问Redis。为了验证Redis应急预案做了一次主备切换演练，切换后将Redis域名从IP-Active修改为IP-Standby，预期应用不需要修改就可以继续连接到Redis(通过IP-Standby)，实际发现应用没有在预期时间内解析到IP-Standby，导致连接Redis失败。

排查过程怀疑是DNS缓存引发的问题，因此对Kubernetes集群中的coreDNS进行了分析，记录下相关信息。

## 2. 问题分析

> 如下代码来自coredns v1.6.7。

### 2.1 coreDNS的cache是如何工作的？

1. 在`ServeDNS()`中处理DNS请求，调用`WriteMsg()`写入响应。
```Go
// /home/kangxiaoning/workspace/coredns/plugin/cache/handler.go

func (c *Cache) ServeDNS(ctx context.Context, w dns.ResponseWriter, r *dns.Msg) (int, error) {
	state := request.Request{W: w, Req: r}

	zone := plugin.Zones(c.Zones).Matches(state.Name())
	if zone == "" {
		return plugin.NextOrFailure(c.Name(), c.Next, ctx, w, r)
	}

	now := c.now().UTC()

	server := metrics.WithServer(ctx)

	ttl := 0
	i := c.getIgnoreTTL(now, state, server)
	if i != nil {
		ttl = i.ttl(now)
	}
	if i == nil || -ttl >= int(c.staleUpTo.Seconds()) {
		crr := &ResponseWriter{ResponseWriter: w, Cache: c, state: state, server: server}
		return plugin.NextOrFailure(c.Name(), c.Next, ctx, crr, r)
	}
	if ttl < 0 {
		servedStale.WithLabelValues(server).Inc()
		// Adjust the time to get a 0 TTL in the reply built from a stale item.
		now = now.Add(time.Duration(ttl) * time.Second)
		go func() {
			r := r.Copy()
			crr := &ResponseWriter{Cache: c, state: state, server: server, prefetch: true, remoteAddr: w.LocalAddr()}
			plugin.NextOrFailure(c.Name(), c.Next, ctx, crr, r)
		}()
	}
	resp := i.toMsg(r, now)
	w.WriteMsg(resp)

	if c.shouldPrefetch(i, now) {
		go c.doPrefetch(ctx, state, server, i, now)
	}
	return dns.RcodeSuccess, nil
}
```
{collapsible="true" collapsed-title="cache.ServeDNS()" default-state="collapsed"}

2. 在`WriteMsg()`写入响应的过程中处理cache，先获取DNS响应消息中的TTL值，再根据cache配置的TTL计算`Duration`，最后当`Duration > 0`时调用`w.set(res, key, mt, duration)`将结果写入cache。

**Duration**表示的是这条消息在cache中的有效时间，默认最小5秒，最大为`Corefile`中配置的TTL。当查询缓存时，根据当前时间、`Duration`可判断缓存是否有效。
```Go
// /home/kangxiaoning/workspace/coredns/plugin/cache/cache.go

func (w *ResponseWriter) WriteMsg(res *dns.Msg) error {
	do := false
	mt, opt := response.Typify(res, w.now().UTC())
	if opt != nil {
		do = opt.Do()
	}

	// key returns empty string for anything we don't want to cache.
	hasKey, key := key(w.state.Name(), res, mt, do)

	msgTTL := dnsutil.MinimalTTL(res, mt)
	var duration time.Duration
	if mt == response.NameError || mt == response.NoData {
		duration = computeTTL(msgTTL, w.minnttl, w.nttl)
	} else if mt == response.ServerError {
		// use default ttl which is 5s
		duration = minTTL
	} else {
		duration = computeTTL(msgTTL, w.minpttl, w.pttl)
	}

	if hasKey && duration > 0 {
		if w.state.Match(res) {
			w.set(res, key, mt, duration)
			cacheSize.WithLabelValues(w.server, Success).Set(float64(w.pcache.Len()))
			cacheSize.WithLabelValues(w.server, Denial).Set(float64(w.ncache.Len()))
		} else {
			// Don't log it, but increment counter
			cacheDrops.WithLabelValues(w.server).Inc()
		}
	}

	if w.prefetch {
		return nil
	}

	// Apply capped TTL to this reply to avoid jarring TTL experience 1799 -> 8 (e.g.)
	ttl := uint32(duration.Seconds())
	for i := range res.Answer {
		res.Answer[i].Header().Ttl = ttl
	}
	for i := range res.Ns {
		res.Ns[i].Header().Ttl = ttl
	}
	for i := range res.Extra {
		if res.Extra[i].Header().Rrtype != dns.TypeOPT {
			res.Extra[i].Header().Ttl = ttl
		}
	}
	return w.ResponseWriter.WriteMsg(res)
}
```
{collapsible="true" collapsed-title="cache.WriteMsg()" default-state="collapsed"}

- `dnsutil.MinimalTTL()`用于获取DNS响应消息中的TTL
```Go
// MinimalTTL scans the message returns the lowest TTL found taking into the response.Type of the message.
func MinimalTTL(m *dns.Msg, mt response.Type) time.Duration {
	if mt != response.NoError && mt != response.NameError && mt != response.NoData {
		return MinimalDefaultTTL
	}

	// No records or OPT is the only record, return a short ttl as a fail safe.
	if len(m.Answer)+len(m.Ns) == 0 &&
		(len(m.Extra) == 0 || (len(m.Extra) == 1 && m.Extra[0].Header().Rrtype == dns.TypeOPT)) {
		return MinimalDefaultTTL
	}

	minTTL := MaximumDefaulTTL
	for _, r := range m.Answer {
		if r.Header().Ttl < uint32(minTTL.Seconds()) {
			minTTL = time.Duration(r.Header().Ttl) * time.Second
		}
	}
	for _, r := range m.Ns {
		if r.Header().Ttl < uint32(minTTL.Seconds()) {
			minTTL = time.Duration(r.Header().Ttl) * time.Second
		}
	}

	for _, r := range m.Extra {
		if r.Header().Rrtype == dns.TypeOPT {
			// OPT records use TTL field for extended rcode and flags
			continue
		}
		if r.Header().Ttl < uint32(minTTL.Seconds()) {
			minTTL = time.Duration(r.Header().Ttl) * time.Second
		}
	}
	return minTTL
}
```
{collapsible="true" collapsed-title="dnsutil.MinimalTTL()" default-state="collapsed"}

- `cache.computeTTL()`用于计算`Duration`
```Go
func computeTTL(msgTTL, minTTL, maxTTL time.Duration) time.Duration {
	ttl := msgTTL
	if ttl < minTTL {
		ttl = minTTL
	}
	if ttl > maxTTL {
		ttl = maxTTL
	}
	return ttl
}
```
{collapsible="true" collapsed-title="cache.computeTTL()" default-state="collapsed"}

- 调用`set()`写入cache
```Go
// /home/kangxiaoning/workspace/coredns/plugin/cache/cache.go

func (w *ResponseWriter) set(m *dns.Msg, key uint64, mt response.Type, duration time.Duration) {
	// duration is expected > 0
	// and key is valid
	switch mt {
	case response.NoError, response.Delegation:
		i := newItem(m, w.now(), duration)
		w.pcache.Add(key, i)

	case response.NameError, response.NoData, response.ServerError:
		i := newItem(m, w.now(), duration)
		w.ncache.Add(key, i)

	case response.OtherError:
		// don't cache these
	default:
		log.Warningf("Caching called with unknown classification: %d", mt)
	}
}
```
{collapsible="true" collapsed-title="cache.set()" default-state="collapsed"}

- 生成Item，将`Duration`写入到`item.origTTL`
```Go
// /home/kangxiaoning/workspace/coredns/plugin/cache/item.go

func newItem(m *dns.Msg, now time.Time, d time.Duration) *item {
	i := new(item)
	i.Rcode = m.Rcode
	i.AuthenticatedData = m.AuthenticatedData
	i.RecursionAvailable = m.RecursionAvailable
	i.Answer = m.Answer
	i.Ns = m.Ns
	i.Extra = make([]dns.RR, len(m.Extra))
	// Don't copy OPT records as these are hop-by-hop.
	j := 0
	for _, e := range m.Extra {
		if e.Header().Rrtype == dns.TypeOPT {
			continue
		}
		i.Extra[j] = e
		j++
	}
	i.Extra = i.Extra[:j]

	i.origTTL = uint32(d.Seconds())
	i.stored = now.UTC()

	i.Freq = new(freq.Freq)

	return i
}
```
{collapsible="true" collapsed-title="cache.newItem()" default-state="collapsed"}

### 2.2 coreDNS会返回过期的cache记录吗？

从下面代码可以看到，在查询cache过程中会判断cache item的TTL是否大于0，大于0表示cache有效，小于0表示cache失效，可以配置失效情况的处理逻辑。

默认情况会重新发起解析，这个结论也可以通过抓包验证。

```Go
// /home/kangxiaoning/workspace/coredns/plugin/cache/handler.go

func (c *Cache) ServeDNS(ctx context.Context, w dns.ResponseWriter, r *dns.Msg) (int, error) {
	state := request.Request{W: w, Req: r}

	zone := plugin.Zones(c.Zones).Matches(state.Name())
	if zone == "" {
		return plugin.NextOrFailure(c.Name(), c.Next, ctx, w, r)
	}

	now := c.now().UTC()

	server := metrics.WithServer(ctx)

	ttl := 0
	i := c.getIgnoreTTL(now, state, server)
	if i != nil {
		ttl = i.ttl(now)
	}
	if i == nil || -ttl >= int(c.staleUpTo.Seconds()) {
		crr := &ResponseWriter{ResponseWriter: w, Cache: c, state: state, server: server}
		return plugin.NextOrFailure(c.Name(), c.Next, ctx, crr, r)
	}
	if ttl < 0 {
		servedStale.WithLabelValues(server).Inc()
		// Adjust the time to get a 0 TTL in the reply built from a stale item.
		now = now.Add(time.Duration(ttl) * time.Second)
		go func() {
			r := r.Copy()
			crr := &ResponseWriter{Cache: c, state: state, server: server, prefetch: true, remoteAddr: w.LocalAddr()}
			plugin.NextOrFailure(c.Name(), c.Next, ctx, crr, r)
		}()
	}
	resp := i.toMsg(r, now)
	w.WriteMsg(resp)

	if c.shouldPrefetch(i, now) {
		go c.doPrefetch(ctx, state, server, i, now)
	}
	return dns.RcodeSuccess, nil
}
```

- 根据当前时间、`item.origTTL`计算缓存条目的TTL
```Go
// /home/kangxiaoning/workspace/coredns/plugin/cache/item.go

func (i *item) ttl(now time.Time) int {
	ttl := int(i.origTTL) - int(now.UTC().Sub(i.stored).Seconds())
	return ttl
}
```

## 3. 分析结论

经过对coreDNS分析，最终判断问题不因为coreDNS引起的，而是外部DNS引起的，结论总结如下。

<procedure>
<img src="nameserver-ttl.svg" alt="etcd service" thumbnail="true"/>
</procedure>
