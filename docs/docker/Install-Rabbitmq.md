

## Compose file



```yaml
version: '3'
services:
  rabbitmq:
    image: harbor.dockerregistry.com/app/rabbitmq:3.8.26-management-alpine
    container_name: "rabbitmq"
    hostname: "rabbitmq"
    ports:
      - 5672:5672    #amqp
      - 15672:15672  #http
      #- 15692:15692  #prometheus
    environment:
      TZ: Asia/Shanghai
      RABBITMQ_DEFAULT_USER: "admin"
      RABBITMQ_DEFAULT_PASS: "admin@123"
    healthcheck:
      test: [ "CMD", "rabbitmqctl", "status"]
      interval: 5s
      timeout: 20s
      retries: 10
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /data/middleware/rabbitmq/data/:/var/lib/rabbitmq/
```



## create_queues.py

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import pika

def create_queue(queue_name):
    credentials = pika.PlainCredentials('admin', 'admin@123')
    parameters = pika.ConnectionParameters('127.0.0.1', 5672, '/', credentials)
    connection = pika.BlockingConnection(parameters)
    channel = connection.channel()

    channel.queue_declare(queue=queue_name, durable=True)

    print("队列 {} 创建成功".format(queue_name))


    connection.close()

def main():
    queue_names = [
        'applyUpdateMessageInformQueue',
        'dataLibQueue',
        'dataServiceUpdateHasApplyQueue',
        'serviceApplyinsertUseServiceQueue',
        'serviceAuthAddQueue',
        'serviceAuthEditQueue',
        'serviceAuthQueue'
    ]  # 要创建的队列名称列表

    for queue_name in queue_names:
        create_queue(queue_name)

if __name__ == '__main__':
    main()
```



## delete_queues.py

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import pika

def delete_queue(queue_name):
    credentials = pika.PlainCredentials('admin', 'admin@123')
    parameters = pika.ConnectionParameters('127.0.0.1', 5672, '/', credentials)
    connection = pika.BlockingConnection(parameters)
    channel = connection.channel()

    channel.queue_delete(queue=queue_name)

    print("队列 {} 删除成功".format(queue_name))

    connection.close()

def main():
    queue_names = [
        'applyUpdateMessageInformQueue',
        'dataLibQueue',
        'dataServiceUpdateHasApplyQueue',
        'serviceApplyinsertUseServiceQueue',
        'serviceAuthAddQueue',
        'serviceAuthEditQueue',
        'serviceAuthQueue'
    ]  # 要删除的队列名称列表

    for queue_name in queue_names:
        delete_queue(queue_name)

if __name__ == '__main__':
    main()
```

