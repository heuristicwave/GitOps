# Monitoring

Metrics Server, Prometheus, Granfana를 차례로 구축하여 EKS 클러스터 모니터링툴을 구축합니다.

## Prometheus

1. [Installing Metrics Server](https://github.com/kubernetes-sigs/metrics-server)

   ```shell
   kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
   ```

2. Set Storage Class - [AWS에서 스토리지 클래스 설정하기](https://kubernetes.io/docs/concepts/storage/storage-classes/#aws-ebs)

3. Installing prometheus with helm

   ```shell
   helm install prometheus prometheus-community/prometheus \
       --namespace monitoring \
       --set alertmanager.persistentVolume.storageClass="gp2" \
       --set alertmanager.persistentVolume.size="512Gi" \
       --set server.persistentVolume.storageClass="gp2" \
       --set server.persistentVolume.size="512Gi"
   ```

   📝 Note Info (localhost:port로 접속해 프로메테우스가 수집한 각종 정보를 확인 할 수 있습니다.)

   ```shell
   Get the Prometheus server URL by running these commands in the same shell:
     export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
     kubectl --namespace monitoring port-forward $POD_NAME 9090

   Get the Alertmanager URL by running these commands in the same shell:
     export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=prometheus,component=alertmanager" -o jsonpath="{.items[0].metadata.name}")
     kubectl --namespace monitoring port-forward $POD_NAME 9093

   Get the PushGateway URL by running these commands in the same shell:
     export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=prometheus,component=pushgateway" -o jsonpath="{.items[0].metadata.name}")
     kubectl --namespace monitoring port-forward $POD_NAME 9091
   ```

4. `monitoring` 네임스페이스 포드 상태 확인

   ```shell
   kubectl get pods -n monitoring
   ```

## Grafana

1. 그라파나가 가져올 Data Source 정의하기

   *url: http://서비스이름.네임스페이스.svc.cluster.local*

   ```yaml
   cat <<EOF > grafana.yaml
   datasources:
     datasources.yaml:
       apiVersion: 1
       datasources:
       - name: Prometheus
         type: prometheus
         url: http://prometheus-server.monitoring.svc.cluster.local
         access: proxy
         isDefault: true
   EOF
   ```

2. `grafana.yaml` 파일의 위치에서 helm으로 그라파나 설치 (확인 : `kubectl get pods -n grafana`)

   ```shell
   helm install grafana grafana/grafana \
       --namespace grafana \
       --set persistence.storageClassName="gp2" \
       --set persistence.size="512Gi" \
       --set persistence.enabled=true \
       --values grafana.yaml \
       --set service.type=LoadBalancer
   ```

3. 📝 Note Info (Grafana 접속 정보)

   ```shell
   1. Get your 'admin' user password by running:
   
      kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
   
   2. The Grafana server can be accessed via port 80 on the following DNS name from within your cluster:
   
      grafana.grafana.svc.cluster.local
   
      Get the Grafana URL to visit by running these commands in the same shell:
   NOTE: It may take a few minutes for the LoadBalancer IP to be available.
           You can watch the status of by running 'kubectl get svc --namespace grafana -w grafana'
        export SERVICE_IP=$(kubectl get svc --namespace grafana grafana -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
        http://$SERVICE_IP:80
   
   3. Login with the password from step 1 and the username: admin
   ```

   > AWS ACM & Route53을 활용해 http 도메인 연결하기 (도메인이 있을 경우)
   >
   > ```shell
   > kubectl annotate -n grafana svc grafana service.beta.kubernetes.io/aws-load-balancer-ssl-cert={acm-arn}
   > kubectl annotate -n grafana svc grafana external-dns.alpha.kubernetes.io/hostname={domain}
   > kubectl annotate -n grafana svc grafana service.beta.kubernetes.io/aws-load-balancer-backend-protocol=http
   > 
   > kubectl patch svc grafana --type=json -p='[{"op": "remove", "path": "/metadata/annotations/prometheus.io~1scrape"}]'
   > ```
   >
   > Grafana manifest를 확인해 보고 싶은 경우
   >
   > ```shell
   > kubectl describe pods -n grafana > grafana-svc.yaml
   > kubectl describe svc -n grafana > grafana-svc.yaml
   > kubectl describe deployment -n grafana > grafana-deployment.yaml
   > kubectl describe replicaset -n grafana > grafana-replicaset.yaml
   > ```

4. Dashboards 만들기

   Manage -> Import -> import -> dashboard url or id 선택 -> 하단 메트릭서버 선택

   > - 1860 - [Node Exporter Full](https://grafana.com/grafana/dashboards/1860)
   > - 6873 - [Analysis by Cluster](https://grafana.com/grafana/dashboards/6873)
   > - 6876 - [Analysis by Namespace](https://grafana.com/grafana/dashboards/6876) (6873 필요)
   > - 6879 - [Analysis by Pod](https://grafana.com/grafana/dashboards/6879)
   > - 10856 - [K8 Cluster Detail Dashboard](https://grafana.com/grafana/dashboards/10856)
   > - 13548 - [Kubernetes / Compute Resources / Node (Groups)](https://grafana.com/grafana/dashboards/13548)
   > - 13770 - [1 Kubernetes All-in-one Cluster Monitoring KR](https://grafana.com/grafana/dashboards/13770)

5. 다른 Data Source 사용해 여러가지 dashboard 만들기

   Configuration -> Data Sources -> Add data source -> CloudWatch

   > - 139 - [AWS Billing](https://grafana.com/grafana/dashboards/139)

<br>

---

참고 자료 : [AWS Documentation](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/prometheus.html), [incredible 블로그](https://incredible.ai/kubernetes/2020/09/08/Prometheus_And_Grafana)
