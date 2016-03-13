一致协议
========

> 注：内容翻译自官网文档 [CONSENSUS PROTOCOL](https://www.consul.io/docs/internals/consensus.html)

Consul使用 [一致协议(consensus protocol)](https://en.wikipedia.org/wiki/Consensus_(computer_science)) 来提供一致性(如在CAP中定义的)。一致协议基于"[Raft: In search of an Understandable Consensus Algorithm](https://ramcloud.stanford.edu/wiki/download/attachments/11370504/raft.pdf)".对于Raft形象的解释，请看[The Secret Lives of Data](The Secret Lives of Data).

	特别备注：上面这个 The Secret Lives of Data 的内容和展现形式都非常的精彩，强烈推荐阅读，绝不要错过！

> 高级主题！这个页面覆盖consul内部的技术细节。有效操作和使用consul并不需要了解这些细节。在这里描述这些细节是为了那些希望了解这些的人，让他们无需深入源代码。