# Домашнее задание к занятию "12.5 Сетевые решения CNI"
После работы с Flannel появилась необходимость обеспечить безопасность для приложения. Для этого лучше всего подойдет Calico.
## Задание 1: установить в кластер CNI плагин Calico


<details>

  <summary>Описание задачи</summary>  
Для проверки других сетевых решений стоит поставить отличный от Flannel плагин — например, Calico. Требования: 
* установка производится через ansible/kubespray;
* после применения следует настроить политику доступа к hello-world извне. Инструкции [kubernetes.io](https://kubernetes.io/docs/concepts/services-networking/network-policies/), [Calico](https://docs.projectcalico.org/about/about-network-policy)

</details>

### Решение

Кластер развернут с использованием kubespray, примеры из предыдущих ДЗ. Плагин Calico установлен по умолчанию.  

Приложение состоит из трех объектов:
- backend
- frontend
- database
  
Сетевое взаимодейсвие возможно:

- frontend > backend
- backend > database
  
Все остальное запрещено.  

В папке templates размещены манифесты на создание объектов.  

1. Разворачиваем deployments и services

```bash
# Развертывание
kubectl apply -f ./templates/main/

# Проверка созданных подов
kubectl get po -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP            NODE    NOMINATED NODE   READINESS GATES
backend-f785447b9-8468p     1/1     Running   0          64m   10.233.90.8   node1   <none>           <none>
database-64747c84f4-4fc8c   1/1     Running   0          62m   10.233.94.9   node0   <none>           <none>
frontend-8645d9cb9c-dnb4c   1/1     Running   0          64m   10.233.90.7   node1   <none>           <none>

``` 
2. Проверяем что на этом этапе нет ограничений по сетевому взаимодействию.

```bash
kubectl exec backend-f785447b9-8468p -- curl -s -m 1 frontend
Praqma Network MultiTool (with NGINX) - frontend-8645d9cb9c-dnb4c - 10.233.90.7

kubectl exec backend-f785447b9-8468p -- curl -s -m 1 backend
Praqma Network MultiTool (with NGINX) - backend-f785447b9-8468p - 10.233.90.8

kubectl exec backend-f785447b9-8468p -- curl -s -m 1 database
Praqma Network MultiTool (with NGINX) - database-64747c84f4-4fc8c - 10.233.94.9

kubectl exec database-64747c84f4-4fc8c -- curl -s -m 1 backend
Praqma Network MultiTool (with NGINX) - backend-f785447b9-8468p - 10.233.90.8
```  

3. Создаем и применяем политику, которая заблокирует все входящие соединения со всех подов:

```bash
# Применяем политику
kubectl apply -f templates/network-policy/00-default.yaml
networkpolicy.networking.k8s.io/default-deny-ingress created

# Проверка соединений
kubectl exec backend-f785447b9-8468p -- curl -s -m 1 frontend
command terminated with exit code 28

kubectl exec backend-f785447b9-8468p -- curl -s -m 1 backend
command terminated with exit code 28

kubectl exec backend-f785447b9-8468p -- curl -s -m 1 database
command terminated with exit code 28

kubectl exec database-64747c84f4-4fc8c -- curl -s -m 1 backend
command terminated with exit code 28
```

3. Применяем остальные сетевые политики и проверяем доступность по требуемой схеме:

```bash
# Применяем политику
kubectl apply -f ./templates/network-policy/
networkpolicy.networking.k8s.io/default-deny-ingress unchanged
networkpolicy.networking.k8s.io/frontend created
networkpolicy.networking.k8s.io/backend created
networkpolicy.networking.k8s.io/database created

# Проверка разрешенных соединений
## frontend > backend
kubectl exec frontend-8645d9cb9c-dnb4c -- curl -s -m 1 backend
Praqma Network MultiTool (with NGINX) - backend-f785447b9-8468p - 10.233.90.8 # успешно

## backend > database
kubectl exec backend-f785447b9-8468p -- curl -s -m 1 database
Praqma Network MultiTool (with NGINX) - database-64747c84f4-4fc8c - 10.233.94.9 # успешно

# Проверка запрещенных соединений
## backend > frontend
kubectl exec backend-f785447b9-8468p -- curl -s -m 1 frontend
command terminated with exit code 28 # запрещено, все ок

## frontend > database
kubectl exec frontend-8645d9cb9c-dnb4c -- curl -s -m 1 database
command terminated with exit code 28 # запрещено, все ок

```


---

## Задание 2: изучить, что запущено по умолчанию


<details>

  <summary>Описание задачи</summary>  

Самый простой способ — проверить командой calicoctl get <type>. Для проверки стоит получить список нод, ipPool и profile.  
Требования:  
* установить утилиту calicoctl;
* получить 3 вышеописанных типа в консоли.  

</details>

### Решение

```bash
# Список нод
calicoctl get node --output wide
NAME    ASN       IPV4             IPV6   
cp0     (64512)   10.128.0.6/24           
node0   (64512)   10.128.0.18/24          
node1   (64512)   10.128.0.8/24   

# ipPool
calicoctl get ipPool --output wide
NAME           CIDR             NAT    IPIPMODE   VXLANMODE   DISABLED   SELECTOR   
default-pool   10.233.64.0/18   true   Never      Always      false      all() 

# profile
calicoctl get profile
NAME                                                 
projectcalico-default-allow                          
kns.default                                          
kns.kube-node-lease                                  
kns.kube-public                                      
kns.kube-system                                      
ksa.default.default                                  
ksa.kube-node-lease.default                          
ksa.kube-public.default                              
ksa.kube-system.attachdetach-controller              
ksa.kube-system.bootstrap-signer                     
ksa.kube-system.calico-kube-controllers              
ksa.kube-system.calico-node                          
ksa.kube-system.certificate-controller               
ksa.kube-system.clusterrole-aggregation-controller   
ksa.kube-system.coredns                              
ksa.kube-system.cronjob-controller                   
ksa.kube-system.daemon-set-controller                
ksa.kube-system.default                              
ksa.kube-system.deployment-controller                
ksa.kube-system.disruption-controller                
ksa.kube-system.dns-autoscaler                       
ksa.kube-system.endpoint-controller                  
ksa.kube-system.endpointslice-controller             
ksa.kube-system.endpointslicemirroring-controller    
ksa.kube-system.ephemeral-volume-controller          
ksa.kube-system.expand-controller                    
ksa.kube-system.generic-garbage-collector            
ksa.kube-system.horizontal-pod-autoscaler            
ksa.kube-system.job-controller                       
ksa.kube-system.kube-proxy                           
ksa.kube-system.namespace-controller                 
ksa.kube-system.node-controller                      
ksa.kube-system.nodelocaldns                         
ksa.kube-system.persistent-volume-binder             
ksa.kube-system.pod-garbage-collector                
ksa.kube-system.pv-protection-controller             
ksa.kube-system.pvc-protection-controller            
ksa.kube-system.replicaset-controller                
ksa.kube-system.replication-controller               
ksa.kube-system.resourcequota-controller             
ksa.kube-system.root-ca-cert-publisher               
ksa.kube-system.service-account-controller           
ksa.kube-system.service-controller                   
ksa.kube-system.statefulset-controller               
ksa.kube-system.token-cleaner                        
ksa.kube-system.ttl-after-finished-controller        
ksa.kube-system.ttl-controller    
```


---