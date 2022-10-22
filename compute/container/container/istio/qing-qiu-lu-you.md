```
$ kubectl get virtualservices -o yaml
- apiVersion: networking.istio.io/v1beta1
  kind: VirtualService
  ...
  spec:
    hosts:
    - details
    http:
    - route:
      - destination:
          host: details
          subset: v1
- apiVersion: networking.istio.io/v1beta1
  kind: VirtualService
  ...
  spec:
    hosts:
    - productpage
    http:
    - route:
      - destination:
          host: productpage
          subset: v1
- apiVersion: networking.istio.io/v1beta1
  kind: VirtualService
  ...
  spec:
    hosts:
    - ratings
    http:
    - route:
      - destination:
          host: ratings
          subset: v1
- apiVersion: networking.istio.io/v1beta1
  kind: VirtualService
  ...
  spec:
    hosts:
    - reviews
    http:
    - route:
      - destination:
          host: reviews
          subset: v1
```

## 基于用户身份的路由 {#route-based-on-user-identity}

```
$ kubectl get virtualservice reviews -o yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
...
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```

在 Bookinfo 应用程序的`/productpage`上，以用户`jason`身份登录。

1. 刷新浏览器。您看到了什么？星级评分显示在每个评论旁边。

2. 以其他用户身份登录（选择您想要的任何名称）。

   刷新浏览器。现在星星消失了。这是因为除了 Jason 之外，所有用户的流量都被路由到`reviews:v1`。

您已成功配置 Istio 以根据用户身份路由流量。

