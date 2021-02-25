# 예제 - 음식배달

표영택 FINAL ASSESSMENT

# 서비스 시나리오

기능적 요구사항
1. 고객이 메뉴와 수량를 선택하여 주문한다.
1. 고객이 주문을 완료하면 주문상태가 "주문됨"로 변경된다.
1. 주문이 완료되면 자동으로 배달이 시작된다.
1. 배달이 시작되면 배달상태가 "배달시작"으로 변경된다.
1. 배달원은 배달을 완료하면 배달상태를 완료로 변경한다.
1. 고객은 주문을 취소할 수 있다.
1. 배달이 완료되면 고객은 주문을 취소할 수 없다.
1. 고객이 주문취소를 하면 주문상태가 "주문취소"로 변경된다.
1. 주문이 취소되면 배달도 취소된다.
2. 주문이 되면 결제를 완료해야 한다.
3. 고객이 주문상태를 중간중간 조회한다.

비기능적 요구사항
1. 트랜잭션
    1. 배달이 취소되지 않은 주문건은 취소되지 않아야 한다.
1. 장애격리
    1. 배달 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다.
1. 성능
    1. 고객은 마이페이지에서 주문 및 배달상태를 조회할 수 있어야 한다.


# 분석/설계

## Event Storming 결과
![Screen Shot 2021-02-24 at 10 25 09 PM](https://user-images.githubusercontent.com/69959639/109115292-32ffa280-7782-11eb-96a6-64e5f86d20aa.png)

## 헥사고날 아키텍처 다이어그램 도출
<img width="985" alt="Screen Shot 2021-02-25 at 4 08 23 PM" src="https://user-images.githubusercontent.com/69959639/109116394-c84f6680-7783-11eb-9df3-8fb1e0b2e8e7.png">

### 비기능 요구사항에 대한 검증

    - 마이크로 서비스를 넘나드는 시나리오에 대한 트랜잭션 처리
        - 고객 주문취소시 배달취소처리: 배달취소가 완료되지 않은 주문취소는 받지 않는다는 경영자의 오랜 신념(?) 에 따라, 
          ACID 트랜잭션 적용. 주문취소시 배달취소에 대해서는 Request-Response 방식 처리
        - 주문 완료시 배송요청 처리: 화면에서 Order 서비스로 주문요청이 전달되는 과정에 있어서 Delivery 서비스가 별도의 배포주기를 가지기 때문에 Eventual Consistency 방식으로 트랜잭션 처리함.

# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 
구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다.


## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다. 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 하지만, 일부 구현에 있어서 영문이 아닌 경우는 실행이 불가능한 경우가 있기 때문에 계속 사용할 방법은 아닌것 같다. (Maven pom.xml, Kafka의 topic id, FeignClient 의 서비스 id 등은 한글로 식별자를 사용하는 경우 오류가 발생하는 것을 확인하였다)

- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다

- 적용 후 REST API 의 테스트
```
# menu 서비스의 메뉴등록처리
http http://a497f79f966814b10ac57259e6fce4ea-1896896990.ap-northeast-2.elb.amazonaws.com:8080/menus menuNm=Gimbab
http http://a497f79f966814b10ac57259e6fce4ea-1896896990.ap-northeast-2.elb.amazonaws.com:8080/menus menuNm=Juice
```
```
# menu 목록 확인
http://a497f79f966814b10ac57259e6fce4ea-1896896990.ap-northeast-2.elb.amazonaws.com:8080/menus

{
  "_embedded" : {
    "menus" : [ {
      "menuNm" : "Gimbab",
      "_links" : {
        "self" : {
          "href" : "http://menu:8080/menus/1"
        },
        "menu" : {
          "href" : "http://menu:8080/menus/1"
        }
      }
    }, {
      "menuNm" : "Juice",
      "_links" : {
        "self" : {
          "href" : "http://menu:8080/menus/2"
        },
        "menu" : {
          "href" : "http://menu:8080/menus/2"
        }
      }
    } ]
  },
  "_links" : {
    "self" : {
      "href" : "http://menu:8080/menus{?page,size,sort}",
      "templated" : true
    },
    "profile" : {
      "href" : "http://menu:8080/profile/menus"
    }
  },
  "page" : {
    "size" : 20,
    "totalElements" : 2,
    "totalPages" : 1,
    "number" : 0
  }
}
```

```
# order 서비스의 주문처리
http http://a497f79f966814b10ac57259e6fce4ea-1896896990.ap-northeast-2.elb.amazonaws.com:8080/orders menuId=1 menuNm=Gimbab qty=1
http http://a497f79f966814b10ac57259e6fce4ea-1896896990.ap-northeast-2.elb.amazonaws.com:8080/orders menuId=2 menuNm=Juice qty=1

# delivery 서비스의 배달처리
http PATCH http://a497f79f966814b10ac57259e6fce4ea-1896896990.ap-northeast-2.elb.amazonaws.com:8080/deliveries/2 status=complete

# order 서비스의 취소처리
http PATCH http://a497f79f966814b10ac57259e6fce4ea-1896896990.ap-northeast-2.elb.amazonaws.com:8080/orders/1 status=cancel

# 주문 및 배달상태 확인
http://a497f79f966814b10ac57259e6fce4ea-1896896990.ap-northeast-2.elb.amazonaws.com:8080/orders
http://a497f79f966814b10ac57259e6fce4ea-1896896990.ap-northeast-2.elb.amazonaws.com:8080/deliveries

{
  "_embedded" : {
    "orders" : [ {
      "menuId" : 1,
      "menuNm" : "Juice",
      "qty" : 1,
      "status" : "cancel",
      "deliveryStatus" : "cancelled",
      "deliveryId" : 1,
      "_links" : {
        "self" : {
          "href" : "http://order:8080/orders/1"
        },
        "order" : {
          "href" : "http://order:8080/orders/1"
        }
      }
    }, {
      "menuId" : 2,
      "menuNm" : "Gimbab",
      "qty" : 2,
      "status" : "ordered",
      "deliveryStatus" : "complete",
      "deliveryId" : 2,
      "_links" : {
        "self" : {
          "href" : "http://order:8080/orders/2"
        },
        "order" : {
          "href" : "http://order:8080/orders/2"
        }
      }
    } ]
  },
  "_links" : {
    "self" : {
      "href" : "http://order:8080/orders{?page,size,sort}",
      "templated" : true
    },
    "profile" : {
      "href" : "http://order:8080/profile/orders"
    }
  },
  "page" : {
    "size" : 20,
    "totalElements" : 2,
    "totalPages" : 1,
    "number" : 0
  }
}

{
  "_embedded" : {
    "deliveries" : [ {
      "orderId" : 1,
      "status" : "cancelled",
      "_links" : {
        "self" : {
          "href" : "http://delivery:8080/deliveries/1"
        },
        "delivery" : {
          "href" : "http://delivery:8080/deliveries/1"
        }
      }
    }, {
      "orderId" : 2,
      "status" : "complete",
      "_links" : {
        "self" : {
          "href" : "http://delivery:8080/deliveries/2"
        },
        "delivery" : {
          "href" : "http://delivery:8080/deliveries/2"
        }
      }
    } ]
  },
  "_links" : {
    "self" : {
      "href" : "http://delivery:8080/deliveries{?page,size,sort}",
      "templated" : true
    },
    "profile" : {
      "href" : "http://delivery:8080/profile/deliveries"
    }
  },
  "page" : {
    "size" : 20,
    "totalElements" : 2,
    "totalPages" : 1,
    "number" : 0
  }
}
```

## API-Gateway

API Gateway를 통하여 동일 진입점으로 진입하여 각 마이크로 서비스를 접근할 수 있다.
```
spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: order
          uri: http://order:8080
          predicates:
            - Path=/orders/**
        - id: delivery
          uri: http://delivery:8080
          predicates:
            - Path=/deliveries/** 
        - id: mypage
          uri: http://mypage:8080
          predicates:
            - Path= /mypages/**
        - id: menu
          uri: http://menu:8080
          predicates:
            - Path=/menus/** 
```

외부에서 접근을 위하여 Gateway의 Service는 LoadBalancer Type으로 생성했다.
```
kubectl expose deploy gateway  --type=LoadBalancer --port=8080
```
```
NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP                                                                  PORT(S)          AGE
service/delivery     ClusterIP      10.100.54.61     <none>                                                                       8080/TCP         43m
service/gateway      LoadBalancer   10.100.203.198   a6ccd4a208aa14d8c8550700879170aa-1276543246.eu-central-1.elb.amazonaws.com   8080:31550/TCP   43m
service/kubernetes   ClusterIP      10.100.0.1       <none>                                                                       443/TCP          6h18m
```

## CQRS / Meterialized View
mypage를 구현하여 order, menu, delivery 서비스의 데이터를 DB Join없이 조회할 수 있다.
```
# mypage 확인
http://a497f79f966814b10ac57259e6fce4ea-1896896990.ap-northeast-2.elb.amazonaws.com:8080/mypages

{
  "_embedded" : {
    "mypages" : [ {
      "orderId" : 1,
      "menuId" : 1,
      "menuNm" : "Juice",
      "deliveryId" : 1,
      "qty" : 1,
      "status" : "cancel",
      "deliveryStatus" : "cancelled",
      "_links" : {
        "self" : {
          "href" : "http://mypage:8080/mypages/1"
        },
        "mypage" : {
          "href" : "http://mypage:8080/mypages/1"
        }
      }
    }, {
      "orderId" : 2,
      "menuId" : 2,
      "menuNm" : "Gimbab",
      "deliveryId" : 2,
      "qty" : 2,
      "status" : "ordered",
      "deliveryStatus" : "complete",
      "_links" : {
        "self" : {
          "href" : "http://mypage:8080/mypages/2"
        },
        "mypage" : {
          "href" : "http://mypage:8080/mypages/2"
        }
      }
    } ]
  },
  "_links" : {
    "self" : {
      "href" : "http://mypage:8080/mypages"
    },
    "profile" : {
      "href" : "http://mypage:8080/profile/mypages"
    },
    "search" : {
      "href" : "http://mypage:8080/mypages/search"
    }
  }
}
```

## Liveness / Readiness 설정
delivery 서비스 deployment.xml 에 Liveness, Readiness를 httpGet 방식으로 설정함
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: delivery
  labels:
    app: delivery
spec:
  replicas: 1
  selector:
    matchLabels:
      app: delivery
  template:
    metadata:
      labels:
        app: delivery
    spec:
      containers:
        - name: delivery
          image: 496278789073.dkr.ecr.eu-central-1.amazonaws.com/skcc20-delivery:v1
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 10
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 10
          livenessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 120
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 5
          resources:
            limits:
              cpu: 500m
            requests:
              cpu: 200m     

```

## Self-Healing (Liveness)

생성된 order pod의 상세정보
```
Name:         delivery-f4b8b7f64-znjtz
Namespace:    default
Priority:     0
Node:         ip-192-168-14-63.eu-central-1.compute.internal/192.168.14.63
Start Time:   Thu, 25 Feb 2021 15:39:48 +0900
Labels:       app=delivery
              pod-template-hash=f4b8b7f64
Annotations:  kubernetes.io/psp: eks.privileged
Status:       Running
IP:           192.168.17.178
IPs:
  IP:           192.168.17.178
Controlled By:  ReplicaSet/delivery-f4b8b7f64
Containers:
  delivery:
    Container ID:   docker://4ae418a192c77e3ae858d04f87f4c89074e73ceca95dec1936844811d16ef8cb
    Image:          496278789073.dkr.ecr.eu-central-1.amazonaws.com/skcc20-delivery:v1
    Image ID:       docker-pullable://496278789073.dkr.ecr.eu-central-1.amazonaws.com/skcc20-delivery@sha256:b1926fbf0133a62767f1975d395fe0d9fb2112792cd64377d9d2ea4812933cbc
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Thu, 25 Feb 2021 15:39:50 +0900
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:  500m
    Requests:
      cpu:        200m
    Liveness:     http-get http://:8080/actuator/health delay=120s timeout=2s period=5s #success=1 #failure=5
    Readiness:    http-get http://:8080/actuator/health delay=10s timeout=2s period=5s #success=1 #failure=10
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-ljz97 (ro)
```

## Readiness

order 서비스가 시작될때 readiness 를 만족할때까지 Ready 상태가 되지 않는것을 확인가능

- deployment.yaml 파일 일부
```
        livenessProbe:
          httpGet:
            path: '/actuator/health'
            port: 8080
          initialDelaySeconds: 120
          timeoutSeconds: 2
          periodSeconds: 5
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: '/actuator/health'    
            port: 8080   
          initialDelaySeconds: 10
          timeoutSeconds: 2
          periodSeconds: 5
          failureThreshold: 5   
```

- 시작 직후
```
NAME                           READY   STATUS    RESTARTS   AGE
pod/order-8466dc99f5-q4c5d     0/1     Running   0          4s
```

- healthy 체크가 정상적으로 이루어진 후
```
NAME                           READY   STATUS    RESTARTS   AGE
pod/order-8466dc99f5-q4c5d     1/1     Running   0          84s
```

## 폴리글랏 퍼시스턴스

앱프런트 (app) 는 H2 대신 HSQL DB를 사용하기로 하였다. 이를 위해 Mypage 서비스의 pom.xml 에는 hsql dependancy를 추가하였다. 그 외 별다른 작업없이 기존의 Entity Pattern 과 Repository Pattern 적용과 데이터베이스 제품의 설정 (application.yml) 만으로 HSQL 에 부착시켰다

- mypage 서비스의 pom.xml 일부

![hsqldb](https://user-images.githubusercontent.com/452079/108929011-973d3c00-7686-11eb-93b2-34f13ebe9006.PNG)


## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 주문취소(order)->배달취소(delivery) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 배달서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 
- 주문취소을 처리하기 직전(@PreUpdate) 배달취소를 요청하도록 처리
- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 배달 시스템이 장애가 나면 주문취소도 못받는다.
```
root@labs--1203739448:/home/project/baedal/myeats/mypage# kubectl delete svc,deploy delivery
service "delivery" deleted
deployment.apps "delivery" deleted

root@labs--1203739448:/home/project/baedal/myeats/mypage# http PATCH http://a497f79f966814b10ac57259e6fce4ea-1896896990.ap-northeast-2.elb.amazonaws.com:8080/orders/3 status=cancel
HTTP/1.1 500 Internal Server Error
Content-Type: application/json;charset=UTF-8
Date: Tue, 23 Feb 2021 08:53:11 GMT
transfer-encoding: chunked

{
    "error": "Internal Server Error", 
    "message": "Could not commit JPA transaction; nested exception is javax.persistence.RollbackException: Error while committing the transaction", 
    "path": "/orders/3", 
    "status": 500, 
    "timestamp": "2021-02-23T08:53:11.602+0000"
}
```
- 배달 시스템이 정상화될 경우 주문취소가 정상적으로 가능하다.
```
root@labs--1203739448:/home/project/baedal/myeats/mypage# http http://a497f79f966814b10ac57259e6fce4ea-1896896990.ap-northeast-2.elb.amazonaws.com:8080/orders menuId=1 menuNm=Gimbab qty=1
HTTP/1.1 201 Created
Content-Type: application/json;charset=UTF-8
Date: Tue, 23 Feb 2021 08:56:35 GMT
Location: http://order:8080/orders/4
transfer-encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://order:8080/orders/4"
        }, 
        "self": {
            "href": "http://order:8080/orders/4"
        }
    }, 
    "deliveryId": null, 
    "deliveryStatus": null, 
    "menuId": 1, 
    "menuNm": "Gimbab", 
    "qty": 1, 
    "status": "ordered"
}

root@labs--1203739448:/home/project/baedal/myeats/mypage# http PATCH http://a497f79f966814b10ac57259e6fce4ea-1896896990.ap-northeast-2.elb.amazonaws.com:8080/orders/1 status=cancel
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Date: Tue, 23 Feb 2021 08:56:59 GMT
transfer-encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://order:8080/orders/1"
        }, 
        "self": {
            "href": "http://order:8080/orders/1"
        }
    }, 
    "deliveryId": 1, 
    "deliveryStatus": "cancelled", 
    "menuId": 1, 
    "menuNm": "Juice", 
    "qty": 1, 
    "status": "cancel"
}
```
- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)


## 비동기식 호출 / 시간적 디커플링 / CB / 장애격리 / 최종 (Eventual) 일관성 테스트

주문이 이루어진 후에 배달시스템으로 알려주는 행위는 동기식이 아니라 비동기식으로 처리하며, 배달 시스템의 처리를 위하여 주문이 블로킹되지 않도록 처리한다.

- 이를 위하여 배달요청이력에 기록을 남긴 후에 곧바로 배달시작이 되었다는 도메인 이벤트를 카프카로 송출한다. (Publish)
- 주문 서비스에서는 배달시작 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다.

배달 시스템은 주문/메뉴와 완전히 분리되어 있으며, 이벤트 수신에 따라 처리되기 때문에, 배달시스팀이 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다.

# 운영

## 디플로이, CI/CD 설정

각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AWS를 사용하였으며, kubectl을 통해 수작업으로 배포하였다.

```

# maven 패키 생성 & docker 이미지 생성 & push
mvn package
docker build -t 496278789073.dkr.ecr.eu-central-1.amazonaws.com/skcc20-order:v2 .
docker push 496278789073.dkr.ecr.eu-central-1.amazonaws.com/skcc20-order:v2

# docker 이미지로 Deployment 생성
kubectl apply -f /home/project/personal/porder/kubernetes/deployment.yml

# expose
kubectl apply -f /home/project/personal/porder/kubernetes/service.yaml
```

## Persistence Volume / Persistence Volume Claim
```
root@labs--1203739448:/home/project/baedal/myeats/order# kubectl exec -it menu-68b9ffcc8-ndrgg -- /bin/sh
/ # df -k
Filesystem           1K-blocks      Used Available Use% Mounted on
overlay               83873772   5671164  78202608   7% /
tmpfs                    65536         0     65536   0% /dev
tmpfs                  1988932         0   1988932   0% /sys/fs/cgroup
/dev/nvme0n1p1        83873772   5671164  78202608   7% /dev/termination-log
127.0.0.1:/          9007199254739968         0 9007199254739968   0% /mnt/aws
/dev/nvme0n1p1        83873772   5671164  78202608   7% /etc/resolv.conf
/dev/nvme0n1p1        83873772   5671164  78202608   7% /etc/hostname
/dev/nvme0n1p1        83873772   5671164  78202608   7% /etc/hosts
shm                      65536         0     65536   0% /dev/shm
tmpfs                  1988932        12   1988920   0% /run/secrets/kubernetes.io/serviceaccount
tmpfs                  1988932         0   1988932   0% /proc/acpi
tmpfs                    65536         0     65536   0% /proc/kcore
tmpfs                    65536         0     65536   0% /proc/keys
tmpfs                    65536         0     65536   0% /proc/latency_stats
tmpfs                    65536         0     65536   0% /proc/timer_list
tmpfs                    65536         0     65536   0% /proc/sched_debug
tmpfs                  1988932         0   1988932   0% /sys/firmware
/ # cd /mnt/aws
/mnt/aws # ls
out1.txt  out2.txt
/mnt/aws # cat out1.txt
Tue Feb 23 00:43:06 UTC 2021
Tue Feb 23 00:43:11 UTC 2021
Tue Feb 23 00:43:16 UTC 2021
Tue Feb 23 00:43:21 UTC 2021
Tue Feb 23 00:43:26 UTC 2021
Tue Feb 23 00:43:31 UTC 2021
Tue Feb 23 00:43:36 UTC 2021
Tue Feb 23 00:43:41 UTC 2021
Tue Feb 23 00:43:46 UTC 2021
(...)
```

### 오토스케일 아웃
사용자 요청이 많아질 때 자동화된 확장 기능을 적용하고자 한다. 

- 배달서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy delivery --min=1 --max=10 --cpu-percent=15
```
- 워크로드를 2분 동안 걸어준다.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://a497f79f966814b10ac57259e6fce4ea-1896896990.ap-northeast-2.elb.amazonaws.com:8080/orders POST {"menuId":"1"}'
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy delivery -w
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:
```
NAME                            READY   STATUS    RESTARTS   AGE
pod/delivery-69d48b98f7-4k7lx   1/1     Running   0          3m9s 
pod/delivery-69d48b98f7-89hfs   1/1     Running   0          3m8s
pod/delivery-69d48b98f7-8c86m   0/1     Pending   0          2m38s
pod/delivery-69d48b98f7-bd5rx   0/1     Pending   0          2m53s
pod/delivery-69d48b98f7-dl6v5   1/1     Running   0          2m53s
pod/delivery-69d48b98f7-drrr7   1/1     Running   0          3m8s
pod/delivery-69d48b98f7-fwvjl   0/1     Pending   0          2m38s
pod/delivery-69d48b98f7-gnq5v   1/1     Running   0          2m53s
pod/delivery-69d48b98f7-r4jc5   1/1     Running   0          2m53s
pod/delivery-69d48b98f7-sp52p   1/1     Running   0          5m4s

(...)

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/delivery   7/10    10           7           5m4s
```

- 다시 어느정도 시간이 지나면 부하가 줄어들면서 다시 스케일 인이 벌어지는것을 확인할 수 있다.
```
NAME                            READY   STATUS    RESTARTS   AGE
pod/delivery-69d48b98f7-4k7lx   1/1     Running   0          13m
pod/delivery-69d48b98f7-sp52p   1/1     Running   0          15m

...(중략)...

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/delivery   2/2     2            2           15m  
```

## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c20 -t120S -v --content-type "application/json" 'http://a2aaaa88c46c04cb2b53ae76248d9d4a-1050609880.ap-northeast-2.elb.amazonaws.com:8080/menus POST {"menuNm":"Coffee"}'

** SIEGE 4.0.2
** Preparing 20 concurrent users for battle.
The server is now under siege...
(...)
HTTP/1.1 500     0.01 secs:     193 bytes ==> POST http://a2aaaa88c46c04cb2b53ae76248d9d4a-1050609880.ap-northeast-2.elb.amazonaws.com:8080/menus
HTTP/1.1 500     0.01 secs:     193 bytes ==> POST http://a2aaaa88c46c04cb2b53ae76248d9d4a-1050609880.ap-northeast-2.elb.amazonaws.com:8080/menus
HTTP/1.1 500     0.01 secs:     193 bytes ==> POST http://a2aaaa88c46c04cb2b53ae76248d9d4a-1050609880.ap-northeast-2.elb.amazonaws.com:8080/menus
HTTP/1.1 201     0.97 secs:     172 bytes ==> POST http://a2aaaa88c46c04cb2b53ae76248d9d4a-1050609880.ap-northeast-2.elb.amazonaws.com:8080/menus
HTTP/1.1 201     1.04 secs:     172 bytes ==> POST http://a2aaaa88c46c04cb2b53ae76248d9d4a-1050609880.ap-northeast-2.elb.amazonaws.com:8080/menus
HTTP/1.1 201     1.12 secs:     174 bytes ==> POST http://a2aaaa88c46c04cb2b53ae76248d9d4a-1050609880.ap-northeast-2.elb.amazonaws.com:8080/menus
(...)
```

- 새버전으로의 배포 시작
```
kubectl set image deployment/order order=496278789073.dkr.ecr.eu-central-1.amazonaws.com/order:v2
```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
```
siege aborted due to excessive socket failure; you
can change the failure threshold in $HOME/.siegerc

Transactions:                    846 hits
Availability:                  44.64 %
Elapsed time:                  38.82 secs
Data transferred:               0.38 MB
Response time:                  1.92 secs
Transaction rate:              21.79 trans/sec
Throughput:                     0.01 MB/sec
Concurrency:                   41.94
Successful transactions:         846
Failed transactions:            1049
Longest transaction:           28.02
Shortest transaction:           0.00

```

배포기간중 Availability 가 평소 100%에서 70% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함:

```
# deployment.yaml 의 readiness probe 의 설정:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: menu
  labels:
    app: menu
spec:
  replicas: 1
  selector:
    matchLabels:
      app: menu
  template:
    metadata:
      labels:
        app: menu
    spec:
      containers:
        - name: t05-menu
          image: 496278789073.dkr.ecr.ap-northeast-2.amazonaws.com/t05-menu:v1
          ports:
          - containerPort: 8080
          readinessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 10
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 10
```
```
yaml 파일을 통한 생성
kubectl apply -f kubernetes/deployment.yaml
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:
```
Lifting the server siege...
Transactions:                  18697 hits
Availability:                 100.00 %
Elapsed time:                 119.04 secs
Data transferred:               3.24 MB
Response time:                  0.38 secs
Transaction rate:             157.06 trans/sec
Throughput:                     0.03 MB/sec
Concurrency:                   60.47
Successful transactions:       18697

```

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.

## AWS 모니터링

AWS CloudWatch 서비스를 통한 모니터링
![aws_moni](https://user-images.githubusercontent.com/452079/108947340-a2529500-76a3-11eb-88e1-c1486704035a.PNG)
