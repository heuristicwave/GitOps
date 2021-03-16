# ALB로 서비스 노출 시키기

alb로 서비스를 노출시키기 과정을 3가지로 나누어 보았다. 각각의 단계를 진행하며 

1. IAM 정책을 생성하고 붙이기
2. Ingress Controller 배포
3. 서비스 NodePort로 노출 후 Ingress 설정

<br>

## 1. IAM

[Policy-document](https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.3/docs/examples/iam-policy.json) 다운받아 `awscli`로 정책 설정하기

```shell
aws iam create-policy --policy-name ALBIngressControllerIAMPolicy --policy-document file:///{$PATH}/iam-policy.json
```

환경변수 세팅

```shell
NG_ROLE=`kubectl -n kube-system describe configmap aws-auth | grep rolearn`
ACCOUNT=${NG_ROLE:24:12}
WN_ROLE=${NG_ROLE:42}
echo "ACCOUNT          : $ACCOUNT"
echo "WORKER NODE ROLE : $WN_ROLE"
echo "NODE GROUP ROLE  : $NG_ROLE"
```

클러스터에 정책 연결

```shell
aws iam attach-role-policy \
--policy-arn arn:aws:iam::${ACCOUNT}:policy/ALBIngressControllerIAMPolicy \
--role-name ${WN_ROLE}
```

<br>

## 2. Ingress Controller

Ingress Controller ClusterRole 생성

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.3/docs/examples/rbac-role.yaml
```

`alb-ingress-controller.yaml` 생성후 확인

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: alb-ingress-controller
  name: alb-ingress-controller
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: alb-ingress-controller
  template:
    metadata:
      labels:
        app.kubernetes.io/name: alb-ingress-controller
    spec:
      containers:
        - name: alb-ingress-controller
          args:
            - --ingress-class=alb
            - --cluster-name={cluster-name}
            - --aws-vpc-id=vpc-{vpc_id}
            - --aws-region={region}
          image: docker.io/amazon/aws-alb-ingress-controller:v1.1.3
      serviceAccountName: alb-ingress-controller
```

`alb-ingress-controller.yaml` 생성후 확인

```shell
kubectl get all -n kube-system
kubectl logs -f deployment.apps/alb-ingress-controller -n kube-system

kubectl get po -A | grep alb	# 전체 pods에서 alb 찾아내기
k logs -n kube-system {alb-ingress-controller name}

# alb-ingress-controller의 로그를 보면 alb 서브넷이 확인되었는지 확인 가능하다.
# 예시
The subnets that did resolve were [\"subnet-{subnet_id}\"]"
```

<br>

## 3. Ingress

ALB로 서비스할 대상 NodePort로 노출하기

```shell
kubectl patch svc {service-name} -n {서비스가 위치한 namespace} -p '{"spec": {"type": "NodePort"}}'
kubectl get svc -n {서비스가 위치한 namespace} | grep {service-name}
```

`ingress.yaml` 생성하고 배포

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: monitoring-ingress
  namespace: {alb를 적용할 namespace}
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/certificate-arn: {acm arn info}
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-2016-08
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80,"HTTPS": 443}]'
    alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'
spec:
  rules:
    - host: {url}
      http:
        paths:
          - backend:
              serviceName: {al를 사용할 svc name}
              servicePort: 80
```

<br>

---

참고 자료 : [FINDA 기술블로그](https://medium.com/finda-tech/aws-eks%EC%97%90%EC%84%9C-service-%ED%83%80%EC%9E%85-%EB%B3%84-ingress-%ED%85%8C%EC%8A%A4%ED%8A%B8-b911f129c8d5), [AUSG 블로그](https://velog.io/@ausg/eks-k8s-elb), [AWS knowledge-center](https://aws.amazon.com/ko/premiumsupport/knowledge-center/eks-alb-ingress-controller-setup/)

