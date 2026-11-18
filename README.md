# CST8915 Lab 7 - CST8915 Full-stack Cloud-native Development: Introduction to Kubernetes Basics

- **Student Name**: Ellie Zheng
- **Student ID**: 041078727
- **Course**: CST8915 Full-stack Cloud-native Development
- **Semester**: Fall 2026

---

## Demo Video

🎥 [Watch Demo Video](https://youtu.be/AWqw92wFSiQ)

---

## Deployment and Verification

For Lab 7, I cleaned up the old Pet Store resources on my existing Lab 6 AKS cluster and deployed `algonquin-pet-store-all-in-one.yaml`. After checking with `kubectl` that all Deployments, Pods, and Services were up, I accessed the Store Front via its public IP, placed a test order, and verified that the message showed up in RabbitMQ.

Next, I deleted the RabbitMQ Pod to test recovery. Kubernetes created a replacement Pod automatically. Before placing any new orders, I checked RabbitMQ again. The management UI showed zero messages, and `rabbitmqctl list_queues name durable messages` didn't list the original `order_queue` anymore, meaning the Pod came back online, but the earlier order message was lost.

---

## RabbitMQ Configuration Analysis

### Is RabbitMQ stateless or stateful?

RabbitMQ is stateful because it needs to retain data like queue definitions and pending messages. For instance, an order message sitting in order_queue has to persist until a consumer service actually picks it up and processes it. See the [RabbitMQ queue documentation](https://www.rabbitmq.com/docs/queues).

### What happens without persistent storage?

In the lab YAML, RabbitMQ runs as a single-replica Deployment without any persistent volume attached. When I deleted the Pod, Kubernetes freshly created a new instance, but because there was no persistent storage mounted, the new Pod had nowhere to pull the previous queue data from. So while the broker came back online, the test order was gone.

Normally, RabbitMQ can recover durable queues and persistent messages after a restart, **as long as the underlying data is still available**. Here, deleting the Pod wiped out its temporary Pod storage, and simply marking a queue as durable won't save data if the storage itself disappears (see [Kubernetes Volumes](https://kubernetes.io/docs/concepts/storage/volumes/) and [RabbitMQ Queues](https://www.rabbitmq.com/docs/queues#durability)).

### How could this be fixed?

One option is to run RabbitMQ on a separate VM and keep its data in a Docker named volume, as we did in Lab 5. The volume stores data outside the container, so a replacement container can reuse its queue data. The VM’s disk must be retained, and the replacement RabbitMQ container must use the same node identity. See [Docker Volumes](https://docs.docker.com/engine/storage/volumes/) and [RabbitMQ Backup and Restore](https://www.rabbitmq.com/docs/backup).

To keep everything inside Kubernetes, a PersistentVolumeClaim (PVC) provides a similar way to keep storage separate from the Pod. RabbitMQ can use a StatefulSet with persistent storage so a replacement Pod can reuse its data, though queues and messages still need to be configured as durable and persistent to survive a restart. See [Kubernetes StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/).

### Would Azure Service Bus solve this problem?

**Yes, for the specific problem shown in this lab.** Since Azure Service Bus is a fully managed message broker, any orders sent to a Service Bus queue are stored externally rather than relying on a RabbitMQ Pod’s local storage inside the AKS cluster. Adopting it would require updating both the Order Service to publish to Service Bus and any consumers to read from it. While this would not recover the test order already lost from RabbitMQ, messages successfully stored in Service Bus would not be lost simply because an AKS Pod was deleted or replaced (see [Microsoft’s Azure Service Bus overview](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview)).
