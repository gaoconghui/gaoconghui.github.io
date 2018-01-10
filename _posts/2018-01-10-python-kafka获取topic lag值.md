---
layout: post
title: kafka-python 获取topic lag值
description: 使用kafka-python获取topic的offset值以及具体group的offset值，从而获取lag值
date: 2018.01.10 21:00 +08:00
tags: 
- python
- kafka
---

说真，这个问题看上去很简单，但“得益”与kafka-python神奇的文档，真的不算简单，反正我是搜了半天还看了半天源码。直接上代码吧

```python
from kafka import SimpleClient, KafkaConsumer
from kafka.common import OffsetRequestPayload, TopicPartition

def get_topic_offset(brokers, topic):
    """
    获取一个topic的offset值的和
    """
    client = SimpleClient(brokers)
    partitions = client.topic_partitions[topic]
    offset_requests = [OffsetRequestPayload(topic, p, -1, 1) for p in partitions.keys()]
    offsets_responses = client.send_offset_request(offset_requests)
    return sum([r.offsets[0] for r in offsets_responses])


def get_group_offset(brokers, group_id, topic):
    """
    获取一个topic特定group已经消费的offset值的和
    """
    consumer = KafkaConsumer(bootstrap_servers=brokers,
                             group_id=group_id,
                             )
    pts = [TopicPartition(topic=topic, partition=i) for i in
           consumer.partitions_for_topic(topic)]
    result = consumer._coordinator.fetch_committed_offsets(pts)
    return sum([r.offset for r in result.values()])


if __name__ == '__main__':
    topic_offset = get_topic_offset("brokers", "topic")
    group_offset = get_group_offset("brokers", "group_id", "topic")
    lag = topic_offset - group_offset
```



