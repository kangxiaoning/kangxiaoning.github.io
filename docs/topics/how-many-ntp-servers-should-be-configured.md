# 应该配置几台NTP Server

直接说答案：至少配置4台NTP Server，详情如下。

## Using Enough Time Sources

An NTP implementation that is compliant with [RFC5905](https://www.rfc-editor.org/rfc/rfc5905.html) takes the available sources of time and submits this timing data to sophisticated intersection, clustering, and combining algorithms to get the best estimate of the correct time.  The description of these algorithms is beyond the scope of this document.  Interested readers should read [RFC5905](https://www.rfc-editor.org/rfc/rfc5905.html) or the detailed description of NTP in [MILLS2006].

- If there is only one source of time, the answer is obvious.  It may not be a good source of time, but it's the only source that can be considered.  Any issue with the time at the source will be passed on to the client.

- If there are two sources of time and they align well enough, then the best time can be calculated easily.  But if one source fails, then the solution degrades to the single-source solution outlined above.  And if the two sources don't agree, it will be difficult to know which one is correct without making use of information from outside of the protocol.

- If there are three sources of time, there is more data available to converge on the best calculated time, and this time is more likely to be accurate.  And the loss of one of the sources (by becoming unreachable or unusable) can be tolerated.  But at that point, the solution degrades to the two-source solution.

- Having four or more sources of time is better as long as the sources are diverse (Section 3.3).  If one of these sources
develops a problem, there are still at least three other time sources.

This analysis assumes that a majority of the servers used in the solution are honest, even if some may be inaccurate.  Operators should be aware of the possibility that if an attacker is in control of the network, the time coming from all servers could be compromised.

Operators who are concerned with maintaining accurate time **SHOULD use at least four independent, diverse sources of time**.  Four sources will provide sufficient backup in case one source goes down.  If four sources are not available, operators MAY use fewer sources, which is subject to the risks outlined above.

Operators are advised to monitor all time sources that are in use. If time sources do not generally align, operators are encouraged to investigate the cause and either correct the problems or stop using defective servers.

# 参考
- [Best practices for NTP](https://access.redhat.com/solutions/778603)