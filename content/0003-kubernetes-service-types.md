---
title: "Kubernetes Service 타입의 종류와 차이점은?"
preview: "Pod는 재시작될 때마다 IP가 바뀝니다. 그런데도 다른 Pod나 외부에서 안정적으로 접근할 수 있는 이유는 뭘까요? Service 타입은 어떻게 나뉘고 무엇이 다를까요?"
tags: [k8s, network]
---

Kubernetes의 Service는 Pod 집합에 대한 안정적인 네트워크 엔드포인트를 제공합니다. Pod는 재시작될 때마다 IP가 바뀌기 때문에 Service 없이는 안정적인 통신이 불가능합니다. Service 타입은 크게 네 가지입니다.

**ClusterIP** (기본값)

클러스터 내부에서만 접근 가능한 가상 IP를 할당합니다. 외부에서는 접근할 수 없고, 같은 클러스터 안의 Pod끼리 통신할 때 사용합니다.

```yaml
spec:
  type: ClusterIP # 생략 가능
  ports:
    - port: 80 # 서비스 노출 포트
      targetPort: 8080 # Pod안의 컨테이너가 리스닝하는 포트
```

**NodePort**

클러스터의 모든 노드에 동일한 포트(30000~32767)를 열어 외부 트래픽을 받습니다. `노드IP:NodePort`로 접근하면 해당 Service의 Pod로 라우팅됩니다. 간단하지만 노드 IP가 노출되고 포트 범위가 제한적이라 프로덕션보다는 테스트 용도로 많이 씁니다.

**LoadBalancer**

클라우드 프로바이더(AWS, GCP, Azure 등)의 로드밸런서를 자동으로 프로비저닝합니다. 외부에서 접근 가능한 IP를 부여받으며, 프로덕션에서 외부 트래픽을 받을 때 가장 일반적으로 사용합니다. 클라우드 환경에서만 동작하고 Service마다 로드밸런서가 생성되어 비용이 발생합니다.

**ExternalName**

클러스터 내부에서 외부 DNS 이름을 추상화할 때 사용합니다. Service 이름으로 접근하면 지정한 외부 도메인으로 CNAME 리다이렉트됩니다. 외부 데이터베이스나 서드파티 API 엔드포인트를 클러스터 내부 이름으로 관리할 때 유용합니다.

```
ClusterIP   → 클러스터 내부 통신
NodePort    → 노드 포트로 외부 접근 (테스트)
LoadBalancer → 클라우드 LB로 외부 접근 (프로덕션)
ExternalName → 외부 DNS 추상화
```

실무에서는 내부 서비스 간 통신은 ClusterIP, 외부 노출은 LoadBalancer를 쓰되, 여러 서비스를 하나의 LB로 관리하려면 Ingress를 함께 사용합니다.

## 참고

- [Kubernetes 공식 문서 — Service](https://kubernetes.io/docs/concepts/services-networking/service/)
