---
layout: post
title: "gRPC in Go: Building High-Performance Service-to-Service APIs"
date: 2026-05-03 12:00:00 +0700
categories: golang backend grpc
tags: [golang, grpc, protobuf, microservices, backend]
description: "Building high-performance service-to-service APIs in Go with gRPC and Protocol Buffers. Covers server streaming, interceptors, retry policies, and domain error mapping."
---

REST is great for public APIs. For internal service-to-service communication, gRPC offers strongly-typed contracts, bidirectional streaming, and significantly better performance.

## Why gRPC Over REST for Internal Services

| Feature | REST+JSON | gRPC+Protobuf |
|---------|-----------|---------------|
| Type safety | Runtime | Compile-time |
| Serialization | ~5-10x slower | Binary, fast |
| Streaming | SSE/WebSocket | Built-in (4 modes) |
| Code generation | Optional | First-class |

## Define Your Contract First

```protobuf
syntax = "proto3";
package order.v1;

service OrderService {
    rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
    rpc GetOrder(GetOrderRequest) returns (Order);
    rpc WatchOrderStatus(WatchRequest) returns (stream OrderStatusEvent);
}

message Order {
    string id = 1;
    string user_id = 2;
    OrderStatus status = 3;
    double total = 4;
}

enum OrderStatus {
    ORDER_STATUS_UNSPECIFIED = 0;
    ORDER_STATUS_PENDING = 1;
    ORDER_STATUS_CONFIRMED = 2;
    ORDER_STATUS_SHIPPED = 3;
}
```

## Implementing the Server

```go
type orderServer struct {
    orderv1.UnimplementedOrderServiceServer
    svc OrderService
}

func (s *orderServer) CreateOrder(ctx context.Context, req *orderv1.CreateOrderRequest) (*orderv1.CreateOrderResponse, error) {
    order, err := s.svc.PlaceOrder(ctx, mapRequestToDomain(req))
    if err != nil {
        if errors.Is(err, ErrInsufficientInventory) {
            return nil, status.Errorf(codes.FailedPrecondition, "insufficient inventory")
        }
        return nil, status.Errorf(codes.Internal, "internal error")
    }
    return &orderv1.CreateOrderResponse{Order: mapDomainToProto(order)}, nil
}

// Server streaming: push status updates to the client
func (s *orderServer) WatchOrderStatus(req *orderv1.WatchRequest, stream orderv1.OrderService_WatchOrderStatusServer) error {
    ch, cancel := s.svc.SubscribeToOrderStatus(stream.Context(), req.OrderId)
    defer cancel()

    for {
        select {
        case <-stream.Context().Done():
            return nil
        case event, ok := <-ch:
            if !ok {
                return nil
            }
            if err := stream.Send(&orderv1.OrderStatusEvent{
                OrderId:   event.OrderID,
                NewStatus: orderv1.OrderStatus(event.Status),
            }); err != nil {
                return err
            }
        }
    }
}
```

## Server with Interceptors

```go
func NewGRPCServer(svc OrderService) *grpc.Server {
    srv := grpc.NewServer(
        grpc.ChainUnaryInterceptor(
            otelgrpc.UnaryServerInterceptor(),
            grpcRecovery.UnaryServerInterceptor(),
        ),
        grpc.KeepaliveParams(keepalive.ServerParameters{
            MaxConnectionIdle: 15 * time.Second,
            Time:              5 * time.Second,
        }),
    )
    orderv1.RegisterOrderServiceServer(srv, &orderServer{svc: svc})
    reflection.Register(srv) // enables grpcurl in dev
    return srv
}
```

## Client with Retry Policy

```go
func NewOrderClient(addr string) (orderv1.OrderServiceClient, error) {
    conn, err := grpc.NewClient(addr,
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithDefaultServiceConfig(`{
            "loadBalancingPolicy": "round_robin",
            "retryPolicy": {
                "MaxAttempts": 3,
                "InitialBackoff": "0.1s",
                "MaxBackoff": "2s",
                "BackoffMultiplier": 2,
                "RetryableStatusCodes": ["UNAVAILABLE"]
            }
        }`),
    )
    if err != nil { return nil, err }
    return orderv1.NewOrderServiceClient(conn), nil
}
```

## Error Mapping

```go
func toGRPCError(err error) error {
    var domainErr *DomainError
    if !errors.As(err, &domainErr) {
        return status.Errorf(codes.Internal, "unexpected error")
    }
    switch domainErr.Code {
    case ErrNotFound:
        return status.Errorf(codes.NotFound, domainErr.Message)
    case ErrInvalidInput:
        return status.Errorf(codes.InvalidArgument, domainErr.Message)
    case ErrConflict:
        return status.Errorf(codes.AlreadyExists, domainErr.Message)
    default:
        return status.Errorf(codes.Internal, "internal error")
    }
}
```

gRPC with Go gives you compile-time type safety across service boundaries — an entire class of runtime bugs simply disappears.
