使用方法：
var (
	oncePool            sync.Once
	instanceRPool       *kelleyRabbimqPool.RabbitPool
	instanceConsumePool *kelleyRabbimqPool.RabbitPool
)

func InitRabbitMq() *kelleyRabbimqPool.RabbitPool {
	oncePool.Do(func() {
		// 初始化生产者
		instanceRPool = kelleyRabbimqPool.NewProductPool()
		// 初始化消费者
		instanceConsumePool = kelleyRabbimqPool.NewConsumePool()
		// 连接 RabbitMQ
		err := instanceRPool.Connect(path, cast.ToInt(port), username, pwd)
		if err != nil {
			fmt.Println(err)
		}

		err = instanceConsumePool.Connect(path, cast.ToInt(port), username, pwd)
		if err != nil {
			panic("监听MQ 消费者失败")
		}
	})
	return instanceRPool
}



// 动态创建多个消费者
func CreateConsumer(exchangeName, queueName, route string, maxReTry int32, autoAck bool) {
	nomrl := &kelleyRabbimqPool.ConsumeReceive{
		ExchangeName: exchangeName,
		ExchangeType: kelleyRabbimqPool.EXCHANGE_TYPE_DIRECT,
		Route:        route,
		QueueName:    queueName,
		IsTry:        true,
		IsAutoAck:    autoAck, // 自动消息确认
		MaxReTry:     maxReTry,
		EventFail: func(code int, e error, data []byte) {
			fmt.Printf("error in consumer for queue %s: %s\n", queueName, e)
		},
		EventSuccess: func(data []byte, header map[string]interface{}, retryClient kelleyRabbimqPool.RetryClientInterface) bool {
			go func(retryClient kelleyRabbimqPool.RetryClientInterface, data []byte) {
				defer retryClient.Ack()
				fmt.Printf("Consumer data for %s: %s\n", queueName, string(data))
			}(retryClient, data)
			return true
		},
	}
	instanceConsumePool.RegisterConsumeReceive(nomrl)
}


// 支持广播类型：
const (
	EXCHANGE_TYPE_FANOUT          = "fanout" //  Fanout：广播，将消息交给所有绑定到交换机的队列
	EXCHANGE_TYPE_DIRECT          = "direct" //Direct：定向，把消息交给符合指定routing key 的队列
	EXCHANGE_TYPE_TOPIC           = "topic"  //Topic：通配符，把消息交给符合routing pattern（路由模式） 的队列
	EXCHANGE_TYPE_DELAYED_MESSAGE = "x-delayed-message"
)
// 初始化多个消费者
func InitConsumers() {
	// 这里我们可以定义一个配置数组，动态创建多个消费者
	consumers := []struct {
		ExchangeName string
		QueueName    string
		Route        string
		MaxReTry     int32  //最大重试次数
		AutoAck      bool   //是否自动确认消息
		MessageType  string // 消息类型（如：订单过期、订单回调）
	}{
		{"test_expired_exchange",kelleyRabbimqPool.EXCHANGE_TYPE_DELAYED_MESSAGE, fmt.Sprintf("%s_queue", testExpired), fmt.Sprintf("%s_router", testExpired), 3, false, testExpired},       // 延期过期
		{"test_exchange",kelleyRabbimqPool.EXCHANGE_TYPE_DIRECT, fmt.Sprintf("%s_queue", testMessage), fmt.Sprintf("%s_router", testMessage), 3, false, testMessage},    // 正常消息
	}

	// 遍历配置数组，创建多个消费者
	for _, consumer := range consumers {
		CreateConsumer(consumer.ExchangeName, consumer.QueueName, consumer.Route, consumer.MaxReTry, consumer.AutoAck)
	}

	// 启动所有消费者
	err := instanceConsumePool.RunConsume()
	if err != nil {
		fmt.Println("Failed to run consumers:", err)
	}
}
