# Dependency Free Transaction Demo
This demo simulates a bare minimal travel application including four services:
* car rental
* flight booking
* hotel reservation
* payment

## Background
A user with travel plans may like making car rental, flight booking, and hotel reservation with just a single service 
instead of three separate services. That's why this application can help.

With microservice architecture, each of the services may have its own database technology and it's not feasible to ensure
all transactions on these services are either committed or rolled back with database. In this demo, we make use of Saga to
ensure eventual data consistency among services.

## Architecture

```
               
               /----> car rental service
User ---> Saga -----> flight booking service
               \----> hotel reservation service
                \---> payment service
```

## Running Demo
1. run the following command to create docker images in saga project root folder.
```
mvn package -DskipTests -Pdocker -Pdemo
```

2. start application up in saga/saga-demo/dependency-free-transaction-demo with the following command
```
docker-compose up
```

## User Requests
A user normally expects to make payment only when transactions with all three services are completed successfully. So Saga
will talk to car rental, flight booking, and hotel reservation services in parallel and then payment service only when the
former three services return success.

The request JSON to ensure this process order looks like the following:
```json
{
  "policy": "BackwardRecovery",
  "requests": [
    {
      "id": "request-car",
      "type": "rest",
      "serviceName": "car-rental-service",
      "transaction": {
        "method": "post",
        "path": "/rentals",
        "params": {
          "form": {
            "customerId": "mike"
          }
        }
      },
      "compensation": {
        "method": "put",
        "path": "/rentals",
        "params": {
          "form": {
            "customerId": "mike"
          }
        }
      }
    },
    {
      "id": "request-hotel",
      "type": "rest",
      "serviceName": "hotel-reservation-service",
      "transaction": {
        "method": "post",
        "path": "/reservations",
        "params": {
          "form": {
            "customerId": "mike"
          }
        }
      },
      "compensation": {
        "method": "put",
        "path": "/reservations",
        "params": {
          "form": {
            "customerId": "mike"
          }
        }
      }
    },
    {
      "id": "request-flight",
      "type": "rest",
      "serviceName": "flight-booking-service",
      "transaction": {
        "method": "post",
        "path": "/bookings",
        "params": {
          "form": {
            "customerId": "mike"
          }
        }
      },
      "compensation": {
        "method": "put",
        "path": "/bookings",
        "params": {
          "form": {
            "customerId": "mike"
          }
        }
      }
    },
    {
      "id": "request-payment",
      "type": "rest",
      "serviceName": "saga-crossapp:payment-service",
      "parents": [
        "request-car",
        "request-flight",
        "request-hotel"
      ],
      "transaction": {
        "method": "post",
        "path": "/payments",
        "params": {
          "form": {
            "customerId": "mike"
          }
        }
      },
      "compensation": {
        "method": "put",
        "path": "/payments",
        "params": {
          "form": {
            "customerId": "mike"
          }
        }
      }
    }
  ]
}
```

The key to the request dependency lies in definition of parents field in the payment request. It means the payment request 
is only executed after `request-car` , `request-flight` , and `request-hotel` are completed. 
```
    "id": "request-payment",
    "type": "rest",
    "serviceName": "payment.servicecomb.io:8080",
    "parents": [
      "request-car",    // request id to car service in the full request json
      "request-flight", // request id to flight service in the full request json
      "request-hotel"   // request id to hotel service in the full request json
    ]
```

To send the above JSON request to Saga, use [postman](https://www.getpostman.com/postman) with POST request to url `http://<docker.host.ip>:8083/requests`

Sending the request more than once will trigger compensation due to insufficient account balance in payment-service.

**Note** transactions and compensations implemented by services must be idempotent. In this demo, we did not enforce that
for simplicity.

To see all events generated by Saga, visit `http://<docker.host.ip>:8083/events` with browser.
