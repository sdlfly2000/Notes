# https://kubesphere.io/zh/blogs/rabbitmq-k8s/

## Update operator image to private repository
kubectl set image deployment/rabbitmq-cluster-operator operator=registry.activator.com/rabbitmqoperator/cluster-operator:2.16.1 -n=rabbitmq-system

## Enter into pod
kubectl exec -it pod/rabbitmq-cluster-server-0 -n operator-rabbitmq -- /bin/bash

## Create K8S Secret
kubectl create secret tls rabbitmq-tls --key default-server-certificate.key --cert default-server-certificate.crt -n operator-rabbitmq

## Upgrade Steps
https://www.rabbitmq.com/docs/rolling-upgrade#example

rabbitmqctl enable_feature_flag all

# Move message to quorum queue

# List shovel
rabbitmqctl list_parameters --vhost /

# On Each Pod
rabbitmq-plugins enable rabbitmq_shovel
rabbitmq-plugins enable rabbitmq_shovel_management

# Create Shovel
rabbitmqctl set_parameter --vhost / shovel migration_to_RmpDomainEvents_shovel '{"src-protocol": "amqp091", "src-uri": "amqp://sdlfly2000@", "src-queue": "ha.queue", "dest-protocol": "amqp091","dest-uri": "amqp://sdlfly2000@/rmp-domain-events","dest-queue": "ha.queue","ack-mode": "on-confirm", "src-delete-after": "never","dest-queue-args": {"x-queue-type": "quorum"}}'

# Clean up Shovel
rabbitmqctl clear_parameter --vhost / shovel migration_to_RmpDomainEvents_shovel

rabbitmq-diagnostics list_policies_with_classic_queue_mirroring
rabbitmq-diagnostics check_if_cluster_has_classic_queue_mirroring_policy