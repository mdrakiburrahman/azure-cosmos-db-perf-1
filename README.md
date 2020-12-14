# Cosmos Optimization Best Practices demo: **Beginner**

This repository contains a beginner overview demo of foundational capabilities in Cosmos DB:

- Identify a partition strategy for your Azure Cosmos DB data.
- Measure the impact of different indexing strategies.

## Setup instructions

1. Follow these [instructions](https://docs.microsoft.com/azure/cosmos-db/how-to-manage-database-account) to create a new Azure Cosmos DB account for SQL API.

2. Clone the repo into an Azure Cloud Shell;

```bash
git clone https://github.com/mdrakiburrahman/azure-cosmos-db-perf-1
cd azure-cosmos-db-perf-1/ExerciseCosmosDB
```

3. Set the environment variables needed for the .NET app to run:

```bash
export COSMOS_NAME=your-cosmos-account
export ENDPOINT=https://your-cosmos-account.documents.azure.com:443/
export KEY=your-cosmos-key

```

## Demo scenarios

### **Scenario 0**: Setup the required databases, containers and import the data.

Run the following scripts in order to create the containers and import the data:

```bash
dotnet run -- -c HotPartition -o InsertDocument -n 20000 -p 10
dotnet run -- -c Orders -o InsertDocument -n 20000 -p 10
```

Note the characteristics of the following parameters:
| Option | Value | Description |
|---|---|---|
| `-c` | `HotPartition` | The name of the collection to use. |
| `-o` | `InsertDocument` | The name of the task to run. |
| `-n` | `20000` | The number of times to run. |
| `-p` | `10` | The degree of parallelism to use. That's the number of threads used for the experiment. The higher this number, the greater the demand on the collection. |

And the container description:

| Container(s) | Notes                                                                                                                      |
| ------------ | -------------------------------------------------------------------------------------------------------------------------- |
| HotPartition | **Throughput:** 7000 RUs <br> **Partition Key:** `/Item.Category` (_hot_), <br> **Indexing**: Default (_consistent, all_). |
| Orders       | **Throughput:** 7000 RUs <br> **Partition Key:** `/Item.id` (_balanced_), <br> **Indexing**: Default (_consistent, all_).  |

---

And a sample data point that's populated - we'll be using for querying:

```json
{
  "OrderTime": "3:18 AM",
  "id": "eb64cadd-3c1b-91b5-9804-0138cd0b5431",
  "OrderStatus": "NEW",
  "Item": {
    "id": "aa2db4e4-80cb-26c2-75bb-219ba57f4c2b",
    "Title": "sd1aiisu3ncykxy",
    "Category": "Electronics",
    "UPC": "66:6019:254",
    "Website": "https://koh.zhsfbim.com",
    "ReleaseDate": "2020-12-13T03:18:46.6118262+00:00",
    "Condition": "NEW",
    "Merchant": "synthesize",
    "ListPricePerItem": 6.59,
    "PurchasePrice": 5.9,
    "Currency": "USD"
  },
  "Quantity": 42,
  "PaymentInstrumentType": 1,
  "PurchaseOrderNumber": "167-42465-08",
  "Customer": {
    "id": "8426d3e6-b49c-eb83-929b-64a1cbca14f6",
    "FirstName": "Liza",
    "LastName": "Hayes",
    "Email": "Liza_Hayes@yahoo.com",
    "StreetAddress": "68268 Sterling Tunnel",
    "ZipCode": "66665-1112",
    "State": "MO"
  },
  "ShippingDate": "2020-12-20T03:18:50.2079941+00:00",
  "Data": "Y98zCVMfGfO4tQ==",
  "_rid": "TK87AP5+FR4BAAAAAAAACA==",
  "_self": "dbs/TK87AA==/colls/TK87AP5+FR4=/docs/TK87AP5+FR4BAAAAAAAACA==/",
  "_etag": "\"3b02c37a-0000-0a00-0000-5fd5881a0000\"",
  "_attachments": "attachments/",
  "_ts": 1607829530
}
```

### **Scenario 1**: Querying across a Single Partition VS Cross Partitions.

Our **Orders** container is partitioned across `/Item.id`.

#### Single Partition Query

Let's run a query that accesses a **single partition** by filtering on `c.Item.id`:

```bash
dotnet run -- -c Orders -o QueryCollection -q "SELECT TOP 1 * FROM c WHERE c.Item.id='aa2db4e4-80cb-26c2-75bb-219ba57f4c2b'"
```

We see ~**2.8 RUs** being consumed:
![image](https://i.imgur.com/BKk52sv.png)

#### Cross Partition Query

Let's grab the same data point, except by accessing **cross partitions** while filtering without the partition key via `c.Customer.id`:

```bash
dotnet run -- -c Orders -o QueryCollection -q "SELECT TOP 1 * FROM c WHERE c.Customer.id='8426d3e6-b49c-eb83-929b-64a1cbca14f6'"
```

We see ~**5.7 RUs** being consumed:
![image](https://i.imgur.com/zg1K4Qj.png)

**Performance**:

We see that querying with the Partition Key as a filter resulted in **<span style="color:green">+100%</span>** improvements in RUs consumed.

**Best practice(s)**:

- Filtering against partition key for read workloads where possible to query across a single partition.

### **Scenario 2**: Bypass the Query Engine to perform a point read.

Our **Orders** container is partitioned across `/Item.id`.

#### Single Partition Query

Let's run a query that accesses a **single partition** by filtering on `c.Item.id`:

```bash
dotnet run -- -c Orders -o QueryCollection -q "SELECT TOP 1 * FROM c WHERE c.Item.id='aa2db4e4-80cb-26c2-75bb-219ba57f4c2b'"
```

We see ~**2.8 RUs** being consumed:
![image](https://i.imgur.com/BKk52sv.png)

#### Cross Partition Query

Reading a document directly from your Azure Cosmos DB collection by using its `id` and `partition key` values is the **least expensive operation**.

Let's grab the same data point above directly from the SDK by specifying the **document id** and **partition key**:

```bash
dotnet run -- -c Orders -o QueryCollection -q "SELECT TOP 1 * FROM c WHERE c.Customer.id='8426d3e6-b49c-eb83-929b-64a1cbca14f6'"
```

We see ~**5.7 RUs** being consumed:
![image](https://i.imgur.com/zg1K4Qj.png)

**Performance**:

We see that querying with the Partition Key as a filter resulted in **<span style="color:green">+100%</span>** improvements in RUs consumed.

**Best practice(s)**:

- Filtering against partition key for read workloads where possible to query across a single partition.
