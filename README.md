# kafka vs rocketmq
- 结论：kafka胜在吞吐量大，rocketmq比kafka提供更丰富的功能，如支持分布式事务、延迟（定时）消息、广播消费、消息过滤、消息轨迹等
<table>
  <tr>
   <td>
   </td>
   <td>kafka
   </td>
   <td>rocketmq
   </td>
  </tr>
  <tr>
   <td>offset提交
   </td>
   <td>默认autoCommit模式，同步提交
   </td>
   <td>默认manualCommit模式，异步提交
   </td>
  </tr>
  <tr>
   <td>消费顺序
   </td>
   <td>默认分区顺序消费。
<p>
支持并发消费，需要指定workerThreads参数
   </td>
   <td>默认并发消费。
<p>
支持分区顺序消费，需要指定ConsumeMode参数，如果消费失败会卡住后面消息（顺序消费不允许跳过，并发消费可以），以固定间隔不断重试消费这个消息。
   </td>
  </tr>
  <tr>
   <td>消息发送
   </td>
   <td>默认批量发送，异步，RT 较长。
   </td>
   <td>默认单条发送，同步，RT 2ms。
   </td>
  </tr>
  <tr>
   <td>消息丢失
   </td>
   <td>会。
<p>
默认autoCommit模式，即atMostOnce，consumer宕机可能导致消息丢失。
<p>
若指定atLeastOnce或manualCommit模式，不会丢失消息。
   </td>
   <td>不会。
<p>
默认manualCommit模式，不会丢失。
   </td>
  </tr>
  <tr>
   <td>消息重复
   </td>
   <td>会
   </td>
   <td>会
   </td>
  </tr>
  <tr>
   <td>消费超时
   </td>
   <td>两次拉取消息的时间间隔超过max.poll.interval.ms（默认5min），broker会认为消费者进程超时，触发rebalance。
<p>
可以通过控制maxPollRecords（每次拉取的数据量，默认100）来缓解这类问题。
   </td>
   <td>并发消费模式下，consumer拉取的一批消息需要在15分钟内消费完毕，超时的消息被当作消费失败，将重新投递，不会丢失。
<p>
顺序消费模式下，没有时间限制。
   </td>
  </tr>
  <tr>
   <td>消息重试（重新消费）
   </td>
   <td>支持，但不友好。
   </td>
   <td>支持
   </td>
  </tr>
  <tr>
   <td>消息重投（重新生产）
   </td>
   <td>支持
   </td>
   <td>支持
   </td>
  </tr>
  <tr>
   <td>回溯消费
   </td>
   <td>支持
   </td>
   <td>支持
   </td>
  </tr>
  <tr>
   <td>消息过滤
   </td>
   <td>不支持
   </td>
   <td>支持。可以指定tag。
   </td>
  </tr>
  <tr>
   <td>广播消费
   </td>
   <td>不支持
   </td>
   <td>支持
   </td>
  </tr>
  <tr>
   <td>限流
   </td>
   <td>不支持
   </td>
   <td>支持。生产者和消费者均可限流，仅支持单机限流。
<p>
生产者默认限流策略，快速失败；消费者默认限流策略，无限等待。
   </td>
  </tr>
  <tr>
   <td>延迟（定时）消息
   </td>
   <td>支持。每条消息指定相同的延时时间。
   </td>
   <td>支持。每条消息指定不同的延时时间，但不保证分区顺序。
   </td>
  </tr>
  <tr>
   <td>exactly once（幂等性）
   </td>
   <td>支持producer在单session单partition的幂等性。
<p>
如果producer宕机再重启/生产跨多个topic-partition，不保证幂等性。
   </td>
   <td>不详
   </td>
  </tr>
  <tr>
   <td>事务消息
   </td>
   <td>支持。
<p>
事务性是一种更强的幂等性，一次事务中需要发送多个消息，要么都发送成功，要么都发送失败。
<p>
支持跨session/跨多个topic-partition的事务性
   </td>
   <td>支持。
<p>
分布式事务场景，本地事务的执行和发消息这两个动作满足事务约束。
   </td>
  </tr>
  <tr>
   <td>manualCommit模式
   </td>
   <td>需要保证最终一定能提交offset，如果consumer抛异常/死循环导致某条消息A的offset无法提交，consumer会继续拉取100批消息消费，此时还未提交消息A的offset，consumer会停止拉取消息。
<p>
autoCommit模式下，只要能拉倒消息，就会提交offset，不管实际消费情况
   </td>
   <td>consumer抛异常，框架会以Later方式提交offset，这种情况下位点能正常提交，但这条消息稍后会重新投递16次。
<p>
consumer死循环导致无法提交offset，会停止拉取消息。
   </td>
  </tr>
  <tr>
   <td>死信队列
   </td>
   <td>不支持
   </td>
   <td>支持
   </td>
  </tr>
  <tr>
   <td>消息轨迹
   </td>
   <td>不支持
   </td>
   <td>支持。根据MessageId或Key到console查询发送和消费的情况。
   </td>
  </tr>
</table>


概念解释：
* 广播消费：相同Consumer Group的每个Consumer实例都接收全量的消息。
* 死信队列：死信队列用于处理无法被正常消费的消息。当一条消息初次消费失败，消息队列会自动进行消息重试；达到最大重试次数后，若消费依然失败，则表明消费者在正常情况下无法正确地消费该消息，此时，消息队列不会立刻将消息丢弃，而是将其发送到该消费者对应的特殊队列中。RocketMQ将这种正常情况下无法被消费的消息称为死信消息（Dead-Letter Message），将存储死信消息的特殊队列称为死信队列（Dead-Letter Queue）。在RocketMQ中，可以通过使用console控制台对死信队列中的消息进行重发来使得消费者实例再次进行消费。
* 消息过滤：一个Topic下面的消息可以具有不同的Tag（一个消息只能有一个Tag），因此可以通过Tag可以将Topic划分为多个子类型，不同的消费者Group可以订阅不同的Tag。
* 延迟队列：broker在接收到延迟消息的时候会把对应延迟级别的消息先存储到对应的延迟队列中，等延迟消息时间到达时，会把消息重新存储到对应的topic的queue里面。
