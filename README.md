# portal-sidecar-summary
※ PaaS-TA Sidecar와 istio sidecar의 이름이 겹치므로 PaaS-TA Sidecar는 이하 문서에서 cf-for-k8s로 언급한다.

### cf-for-k8s http 사용
- 배포된 gateway에서 http 리다이렉트 되는 설정을 false로 설정한다
```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-ingressgateway
  namespace: cf-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - '*'
    port:
      name: http
      number: 80
      protocol: HTTP
    tls:
      httpsRedirect: false
      ..................
      ..................
```
- 혹은 manifest/sidecar-values.yml에 해당 내용을 추가해서 rendering을 진행한다.
```diff
is_self_signed_certificate: true
+gateway:
+  https_only: false

```


### cf-for-k8s variable
- 편안한 Portal APP manifest 설정을 위한 manifest/sidecar-values.yml 의 변경이 필요 하다.
```
cf_admin_password: ******

cf_db:
  admin_password: ******
  
capi:
  database:
    password: ******
    
uaa:
  database:
    password: ******
  admin_client_secret: ******
  login_secret: ******

```

### k8s networkpolicy
- ~~외부의 portal db를 설정 할 시 create-security-group을 이용하여 portal 외부 db를 등록해야한다.~~ (재 확인 결과 적용 안해도 정상동작하는거로 보임.. 추후 수정 예정)
```
$ vi portal_rule.json
[
    {
        "destination": "10.0.2.120",
        "protocol": "all"
    }
]

$ cf create-security-group portal portal_rule.json
$ cf bind-runnning-security-group portal
```

- cf-db와 Portal app이 통신하기 위하여 networkpolicy를 등록한다
```
$ vi allow-cf-db-ingress-from-cf-workloads.yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-cf-db-ingress-from-cf-workloads
  namespace: cf-db
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          cf-for-k8s.cloudfoundry.org/cf-workloads-ns: ""
      podSelector:
        matchLabels:
          cloudfoundry.org/org_name: system # portal org 이름
      podSelector:
        matchLabels:
          cloudfoundry.org/space_name: dev # portal space 이름
---
$ kubectl apply -f allow-cf-db-ingress-from-cf-workloads.yaml
```

### swfit object stroage endpoint 설정
- k8s의 service를 이용하여 외부 swift object storage와 연결 할 시 swift의 endpoint를 수정해야 한다.
```
# swift가 설치된 환경의 openstack 정보를 export 한다.
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://localhost:14999/v3
export OS_IDENTITY_API_VERSION=3

# openstack cli를 이용하여 endpoint를 확인한다.
$ openstack
$ endpoint list

# endpoint 의 url이 사용하려는 경우와 다른경우 수정/생성한다. (list를 확인 후 내용을 수정한다)
$ endpoint create --region paasta swift public http://storage-service.paasta.svc.cluster.local:10008/v1/AUTH_%\(project_id\)s
$ endpoint create --region paasta keystone  public http://storage-service.paasta.svc.cluster.local:14999/v3/ # keystone이 중복일 경우 ID로 설정


```

### istio(sidecar)
- cf-workloads가 다른 k8s service와 통신을 하려면 k8s sidecar의 설정을 약간 변경해줘야 한다.
```
$ vi portal-sidecar.yml
---
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: portal-db
  namespace: cf-workloads
spec:
  workloadSelector:
    labels:
      cloudfoundry.org/org_name: system   # portal app 이 배포될 org name 
      cloudfoundry.org/space_name: dev    # portal app 이 배포될 space name
  egress:
  - hosts:
    - cf-db/*    # portal-app 이 cf-db와 통신 가능하게 설정
    - ......     # 필요 시 추가 설정
```

- 최종적으로는 Portal DB와 Storage가 POD로 작동될 예정 이지만 테스트 용으로 외부 infra와 통신하게 설정하였다.
```
$ vi storage-service.yml
apiVersion: v1
kind: Service
metadata:
  name: storage-service
  namespace: paasta
spec:
  ports:
    - port: 15001
      targetPort: 15001
      name : keystone
    - port: 10008
      targetPort: 10008
      name : proxy
    - port: 13306
      targetPort: 13306
      name : mariadb
      appProtocol: mysql # istio의 경우 특정 응용프로그램의 protocol을 정확히 명시해야 한다.
---
apiVersion: v1
kind: Endpoints
metadata:
  name: storage-service
  namespace: paasta
subsets:
- addresses:
    - ip: 10.0.2.120
  ports:
    - port: 15001
      name : keystone
    - port: 10008
      name : proxy
    - port: 13306
      name0 : mariadb
      
## 설정 후 sidecar의 egress 변경 필요
```



### portal client
- 기본 Sidecar를 배포할 시 portalclient가 미 생성 되있으므로 uaac로 uaa에 접근하여 portalclient를 세팅해야 한다.

```
SYSTEM_DOMAIN=********
APP_DOMAIN=********
UAA_ADMIN_ID=********
UAA_ADMIN_SECRET********

uaac target uaa.$SYSTEM_DOMAIN --skip-ssl-validation
uaac token client get $UAA_ADMIN_ID -s $UAA_ADMIN_SECRET

uaac client add portalclient -s clientsecret --redirect_uri "http://portal-web-user.$APP_DOMAIN, http://portal-web-user.$APP_DOMAIN/callback" \
--scope "cloud_controller_service_permissions.read , openid , cloud_controller.read , cloud_controller.write , cloud_controller.admin" \
--authorized_grant_types "authorization_code , client_credentials , refresh_token" \
--authorities="uaa.resource" \
--autoapprove="openid , cloud_controller_service_permissions.read"
```
