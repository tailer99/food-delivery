# 예제 - 음식배달

5팀 FINAL ASSESSMENT

# 서비스 시나리오

기능적 요구사항
1. 고객이 메뉴와 수량를 선택하여 주문한다.
1. 고객이 주문을 완료하면 주문상태가 "주문완료"로 변경된다.
1. 주문이 완료되면 자동으로 배달이 시작된다.
1. 배달이 시작되면 배달상태가 "배달시작"으로 변경된다.
1. 배달원은 배달을 완료하면 배달상태를 완료로 변경한다.
1. 고객은 주문을 취소할 수 있다.
1. 배달이 완료되면 고객은 주문을 취소할 수 없다.
1. 고객이 주문취소를 하면 주문상태가 "주문취소"로 변경된다.
1. 주문이 취소되면 배달도 취소된다.
1. 고객이 주문상태를 중간중간 조회한다.

비기능적 요구사항
1. 트랜잭션
    1. 배달이 취소되지 않은 주문건은 취소되지 않아야 한다.
1. 장애격리
    1. 배달 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다.
    1. 배달 시스템이 과중되면 주문을 잠시동안 받지 않고 잠시후에 하도록 유도한다.
1. 성능
    1. 고객은 마이페이지에서 주문 및 배달상태를 조회할 수 있어야 한다.


# 분석/설계

## Event Storming 결과
![이미지](https://user-images.githubusercontent.com/452079/108806757-eda66e00-75e5-11eb-920f-6dc60e76d44a.PNG)

## 헥사고날 아키텍처 다이어그램 도출
![hexa](https://user-images.githubusercontent.com/452079/108806765-f39c4f00-75e5-11eb-8eb8-caae37dedf31.PNG)

### 비기능 요구사항에 대한 검증

    - 마이크로 서비스를 넘나드는 시나리오에 대한 트랜잭션 처리
        - 고객 주문취소시 배달취소처리: 배달취소가 완료되지 않은 주문취소는 받지 않는다는 경영자의 오랜 신념(?) 에 따라, 
          ACID 트랜잭션 적용. 주문취소시 배달취소에 대해서는 Request-Response 방식 처리
        - 주문 완료시 배송요청 처리: 화면에서 Order 서비스로 주문요청이 전달되는 과정에 있어서 Delivery 서비스가 별도의 배포주기를 가지기 때문에 Eventual Consistency 방식으로 트랜잭션 처리함.

# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 
구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)


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
  profiles: default
  cloud:
    gateway:
      routes:
        - id: order
          uri: http://localhost:8081
          predicates:
            - Path=/orders/**
        - id: delivery
          uri: http://localhost:8082
          predicates:
            - Path=/deliveries/** 
        - id: mypage
          uri: http://localhost:8083
          predicates:
            - Path= /mypages/**
        - id: menu
          uri: http://localhost:8084
          predicates:
            - Path=/menus/** 
```

외부에서 접근을 위하여 Gateway의 Service는 LoadBalancer Type으로 생성했다.
```
kubectl expose deploy gateway  --type=LoadBalancer --port=8080
```
```
NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP                                 PORT(S)          AGE
service/gateway      LoadBalancer   10.100.208.95   censored.ap-northeast-2.elb.amazonaws.com   8080:30302/TCP   2s
service/kubernetes   ClusterIP      10.100.0.1      <none>                                      443/TCP          108m
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
order 서비스 deployment.xml 에 Liveness, Readiness를 httpGet 방식으로 설정함
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order
  labels:
    app: order
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order
  template:
    metadata:
      labels:
        app: order
    spec:
      containers:
        - name: order
          image: 496278789073.dkr.ecr.ap-northeast-2.amazonaws.com/team05-order:v1
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

```

## Self-Healing

order pod를 강제종료처리시, liveness에 의해 자동으로 다른 pod가 생성되는 모습
```
kubectl delete pod order-cb5d6b495-knfrp
```
![image](https://user-images.githubusercontent.com/452079/108833009-974e2500-760f-11eb-99c2-0e4b127f245d.png)

생성된 order pod의 상세정보
```
Containers:
  order:
    Container ID:   docker://0cafff76bed772615cfe8b74845e50f3979fec7fe5da5a0587b7ae2b4fe79b4e
    Image:          496278789073.dkr.ecr.ap-northeast-2.amazonaws.com/team05-order:v1
    Image ID:       docker-pullable://496278789073.dkr.ecr.ap-northeast-2.amazonaws.com/team05-order@sha256:8b09186218c6b2517e8829cdf48decfd105d10831c2870677b29f50adfcbc61a
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Tue, 23 Feb 2021 19:38:50 +0900
    Ready:          True
    Restart Count:  0
    Liveness:       http-get http://:8080/actuator/health delay=120s timeout=2s period=5s #success=1 #failure=5
    Readiness:      http-get http://:8080/actuator/health delay=10s timeout=2s period=5s #success=1 #failure=10
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-c55nd (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True
```

## 폴리글랏 퍼시스턴스

앱프런트 (app) 는 H2 대신 HSQL DB를 사용하기로 하였다. 이를 위해 Mypage 서비스의 pom.xml 에는 hsql dependancy를 추가하였다. 그 외 별다른 작업없이 기존의 Entity Pattern 과 Repository Pattern 적용과 데이터베이스 제품의 설정 (application.yml) 만으로 HSQL 에 부착시켰다

- mypage 서비스의 pom.xml 일부

![hexa2](https://user-images.githubusercontent.com/452079/108807227-2bf05d00-75e7-11eb-8242-1588cc74d3a8.PNG)


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


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트

주문이 이루어진 후에 배달시스템으로 알려주는 행위는 동기식이 아니라 비동기식으로 처리하며, 배달 시스템의 처리를 위하여 주문이 블로킹되지 않도록 처리한다.

- 이를 위하여 배달요청이력에 기록을 남긴 후에 곧바로 배달시작이 되었다는 도메인 이벤트를 카프카로 송출한다. (Publish)
- 주문 서비스에서는 배달시작 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다.

배달 시스템은 주문/메뉴와 완전히 분리되어 있으며, 이벤트 수신에 따라 처리되기 때문에, 배달시스팀에 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다.


# 운영

## CI/CD 설정

각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AWS를 사용하였으며, kubectl을 통해 수작업으로 배포하였다.

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

## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 단말앱(app)-->결제(pay) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml

hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```

- 피호출 서비스(배달:delivery) 의 임의 부하 처리 - 800 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
    # (order) 주문
    @PrePersist
    public void onPrePersist(){  // 배달을 처리한 뒤 임의부하
        try {
            Thread.currentThread().sleep((long) (800 + Math.random() * 220));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시

```
$ siege -c100 -t60S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"item": "chicken"}'

** SIEGE 4.0.5
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.73 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.75 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.77 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.97 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.81 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.87 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.12 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.16 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.17 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.26 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.25 secs:     207 bytes ==> POST http://localhost:8081/orders

* 요청이 과도하여 CB를 동작함 요청을 차단

HTTP/1.1 500     1.29 secs:     248 bytes ==> POST http://localhost:8081/orders   
HTTP/1.1 500     1.24 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.23 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.42 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     2.08 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.29 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.24 secs:     248 bytes ==> POST http://localhost:8081/orders

* 요청을 어느정도 돌려보내고나니, 기존에 밀린 일들이 처리되었고, 회로를 닫아 요청을 다시 받기 시작

HTTP/1.1 201     1.46 secs:     207 bytes ==> POST http://localhost:8081/orders  
HTTP/1.1 201     1.33 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.36 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.63 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.65 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.69 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.71 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.71 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.74 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.76 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.79 secs:     207 bytes ==> POST http://localhost:8081/orders

* 다시 요청이 쌓이기 시작하여 건당 처리시간이 610 밀리를 살짝 넘기기 시작 => 회로 열기 => 요청 실패처리

HTTP/1.1 500     1.93 secs:     248 bytes ==> POST http://localhost:8081/orders    
HTTP/1.1 500     1.92 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.93 secs:     248 bytes ==> POST http://localhost:8081/orders

* 생각보다 빨리 상태 호전됨 - (건당 (쓰레드당) 처리시간이 610 밀리 미만으로 회복) => 요청 수락

HTTP/1.1 201     2.24 secs:     207 bytes ==> POST http://localhost:8081/orders  
HTTP/1.1 201     2.32 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.16 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.19 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.19 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.19 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.21 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.29 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.30 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.38 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.59 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.61 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.62 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.64 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.01 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.27 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.33 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.45 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.52 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.57 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.69 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.69 secs:     207 bytes ==> POST http://localhost:8081/orders

* 이후 이러한 패턴이 계속 반복되면서 시스템은 도미노 현상이나 자원 소모의 폭주 없이 잘 운영됨


HTTP/1.1 500     4.76 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.23 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.76 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.74 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.82 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.82 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.84 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.66 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     5.03 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.22 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.19 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.18 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.69 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.65 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     5.13 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.84 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.25 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.25 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.80 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.87 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.33 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.86 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.96 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.34 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.04 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.50 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.95 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.54 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.65 secs:     207 bytes ==> POST http://localhost:8081/orders


:
:

Transactions:		        1025 hits
Availability:		       63.55 %
Elapsed time:		       59.78 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02
Successful transactions:        1025
Failed transactions:	         588
Longest transaction:	        9.20
Shortest transaction:	        0.00

```
- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만, 63.55% 가 성공하였고, 46%가 실패했다는 것은 고객 사용성에 있어 좋지 않기 때문에 Retry 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.

- Retry 의 설정 (istio)
- Availability 가 높아진 것을 확인 (siege)

### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 

- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy delivery --min=1 --max=10 --cpu-percent=15
```
- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
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
siege -c100 -t120S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"item": "chicken"}'

** SIEGE 4.0.5
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
:

```

- 새버전으로의 배포 시작
```
kubectl set image ...
```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
```
Transactions:		        3078 hits
Availability:		       70.45 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02

```
배포기간중 Availability 가 평소 100%에서 70% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함:

```
# deployment.yaml 의 readiness probe 의 설정:


kubectl apply -f kubernetes/deployment.yaml
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:
```
Transactions:		        3078 hits
Availability:		       100 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02

```

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.

