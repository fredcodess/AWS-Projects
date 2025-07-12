# A fully serverless CRUD TODO API (AWS Lambda + API Gateway + DynamoDB)

---

## 📌 Objectives

* Build a simple but complete REST API (CRUD) with zero servers.
* Implement scalable backend logic with AWS Lambda and API Gateway.
* Securely access and manipulate data in DynamoDB using IAM roles.
* Connect HTTP endpoints to backend logic with minimal infrastructure overhead.

---

## ✅ Features

* **Serverless**: No EC2 or containers. All resources scale automatically.
* **CRUD support**: Create, Read, Update, Delete todos.
* **Unique IDs**: Items use a simple hardcoded UUID generator in Lambda.
* **CORS enabled**: Ready to integrate with any frontend.
* **Free tier–friendly**: Uses minimal resources.

---

## 🔗 API Endpoints

| Method | Endpoint      | Description             |
| ------ | ------------- | ----------------------- |
| GET    | `/todos`      | Get all todos           |
| GET    | `/todos/{id}` | Get a todo by ID        |
| POST   | `/todos`      | Create a new todo       |
| PUT    | `/todos/{id}` | Update an existing todo |
| DELETE | `/todos/{id}` | Delete a todo           |

Each todo has the shape:

```json
{
  "id": "abc123xyz",
  "title": "Buy milk",
  "done": false
}
```

---

## 🚀 How to Deploy

Follow these steps to deploy the backend entirely within AWS.

### 1. Create DynamoDB Table

* Name: `Todos`
* Partition key: `id` (String)

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/dynamodb-table.png?raw=true)

---

### 2. Create Lambda Function

* Runtime: Node.js 18
* Add handler code.
* Set environment variable: `TABLE=Todos`

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/lambda-function.png?raw=true)

---

### 3. Attach IAM Role to Lambda

* Attach policy: `AmazonDynamoDBFullAccess` (or use least privilege in production)

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/iam-role-attach.png?raw=true)


---

### 4. Create API Gateway

* REST API or HTTP API (REST recommended for full control)
* Add resource: `/todos`
* Add methods: `GET`, `POST`
* Add child resource: `/todos/{id}`
* Add methods: `GET`, `PUT`, `DELETE`
* Integrate each with Lambda
* Enable **CORS** on each method

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/api-methods.png?raw=true)

---

### 5. Deploy API

* Create a new **stage**: `prod`
* Deploy your API

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/api-deployment.png?raw=true)

---

### 6. Test the API

You can test endpoints with **Postman**, **curl**, or your frontend.

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/postman-tests.png?raw=true)

