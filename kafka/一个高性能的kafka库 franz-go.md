## 一个高性能的kafka库 franz-go

---

- 导入包
    ```azure
    go get github.com/twmb/franz-go
    ```

- 使用示例
    ```go
    seeds := []string{"localhost:9092"}
    // One client can both produce and consume!
    // Consuming can either be direct (no consumer group), or through a group. Below, we use a group.
    cl, err := kgo.NewClient(
        kgo.SeedBrokers(seeds...),
        kgo.ConsumerGroup("my-group-identifier"),
        kgo.ConsumeTopics("foo"),
    )
    if err != nil {
        panic(err)
    }
    defer cl.Close()
    
    ctx := context.Background()
    
    // 1.) Producing a message
    // All record production goes through Produce, and the callback can be used
    // to allow for synchronous or asynchronous production.
    var wg sync.WaitGroup
    wg.Add(1)
    record := &kgo.Record{Topic: "foo", Value: []byte("bar")}
    cl.Produce(ctx, record, func(_ *kgo.Record, err error) {
        defer wg.Done()
        if err != nil {
            fmt.Printf("record had a produce error: %v\n", err)
        }
    
    })
    wg.Wait()
    
    // Alternatively, ProduceSync exists to synchronously produce a batch of records.
    if err := cl.ProduceSync(ctx, record).FirstErr(); err != nil {
        fmt.Printf("record had a produce error while synchronously producing: %v\n", err)
    }
    
    // 2.) Consuming messages from a topic
    for {
        fetches := cl.PollFetches(ctx)
        if errs := fetches.Errors(); len(errs) > 0 {
            // All errors are retried internally when fetching, but non-retriable errors are
            // returned from polls so that users can notice and take action.
            panic(fmt.Sprint(errs))
        }
    
        // We can iterate through a record iterator...
        iter := fetches.RecordIter()
        for !iter.Done() {
            record := iter.Next()
            fmt.Println(string(record.Value), "from an iterator!")
        }
    
        // or a callback function.
        fetches.EachPartition(func(p kgo.FetchTopicPartition) {
            for _, record := range p.Records {
                fmt.Println(string(record.Value), "from range inside a callback!")
            }
    
            // We can even use a second callback!
            p.EachRecord(func(record *kgo.Record) {
                fmt.Println(string(record.Value), "from a second callback!")
            })
        })
    }
    ```