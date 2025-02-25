# 1. Let's Discuss the Problem - DB can perform less OPS - has low throughput.
## **The Problem: Real-Time Driver or Delivery Boy Tracking in Zomato/Uber**

In apps like **Uber** (for drivers) and **Zomato** (for delivery agents), one of the biggest technical challenges is **real-time tracking**. This means:

- Customers should see the **exact location** of their driver/delivery agent.
- The app should update **instantly** when the driver moves.
- The restaurant or driver should also get **live updates** about order pickup and drop-off.

If this tracking is **delayed, inaccurate, or slow**, it leads to:
❌ A bad customer experience.  
❌ Drivers missing important updates.  
❌ Inefficient route optimization.  

## **Why is This Problem Difficult?**

### **1. High Volume of Location Updates** 📍
- Every driver/delivery person sends **new GPS coordinates every few seconds**.
- If 100,000 drivers are online, that’s **100,000 location updates per second** (6 million updates per minute)!

### **2. Real-Time Processing is Required** ⚡
- Location data must be *processed* and *displayed* **instantly**.
- Traditional databases (like PostgreSQL or MySQL) are too slow for this.
- They can't handle so many operations(Read and Write) per second.

### **3. Multiple Services Need the Data** 🔄
- **Customers** need to see the driver’s live location.
- **Logistics team** needs real-time route optimization.
- **Restaurant** needs to know when the driver is arriving.
- **Notification system** needs to send alerts (e.g., "Your delivery agent is near").

A slow or inefficient system can cause major issues like:
- A driver has reached the destination, but the app still shows them far away.
- A delivery agent picks up the food, but the restaurant doesn’t get an update.
- The customer doesn’t get notifications, leading to confusion.

---

# 2. Kafka - The Solution

- Kafka has **⚡high throughput** but **🔻temporary storage.**
- Query not possible in kafka.

# 🚀  Want to See Kafka in Action? - Node.js Kafka App for Real-Time Driver Location Tracking
![image](https://github.com/user-attachments/assets/2666bb77-67e6-440a-a4b9-0f57ea2cda71)
## Zomato Rider Location Update Architecture using Kafka

This architecture ensures real-time rider location updates using WebSockets, Kafka, and Consumer Groups.

### 1️⃣ Flow of Data (Step-by-Step)

#### Step 1: Client (Driver) Sends Location Updates
- The driver's mobile app continuously sends location updates via WebSocket.
- The updates contain:

```json
{
  "driverId": 453,
  "longitude": 1324234.234,
  "latitude": 4524234.234,
  "state": "south"
}
```

- The Backend (BE) maintains an open WebSocket connection with the driver.

#### Step 2: Backend Produces Kafka Messages
- The backend Kafka Producer processes incoming location updates.
- The producer publishes messages to the Kafka topic **"rider_updates"**.
- **Partitioning Logic:**
  - If `state === "south"`, the message is sent to **Partition 0 (p0)**.
  - Otherwise, it is sent to **Partition 1 (p1)**.

#### Step 3: Kafka Handles the Messages
- Kafka stores messages in a partitioned topic (**rider_updates**).
- Partitions ensure parallel processing and high throughput.
- Kafka guarantees ordering **within each partition**.

#### Step 4: Consumer Groups Process the Messages
- Consumers subscribe to the **"rider_updates"** topic.
- Consumers read from partitions:
  - One consumer can read multiple partitions.
  - **Multiple consumers in a group do NOT share partitions.**
- Possible consumers include:
  - Customer App (To track the driver's location in real time).
  - Restaurant Dashboard (To estimate delivery times).
  - Support Team (For troubleshooting delivery issues).
  - Database Service (Batch inserts data into a database for historical tracking).
![image](https://github.com/user-attachments/assets/163535b6-07f8-46da-8344-d69d707f85b0)

# CODE


## Commands
- Start Zookeper Container and expose PORT `2181`.
```bash
docker run -p 2181:2181 zookeeper
```
- Start Kafka Container, expose PORT `9092` and setup ENV variables.
```bash
docker run -p 9092:9092 \
-e KAFKA_ZOOKEEPER_CONNECT=<PRIVATE_IP>:2181 \
-e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://<PRIVATE_IP>:9092 \
-e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
confluentinc/cp-kafka
```

## Code
`client.js`
```js
const { Kafka } = require("kafkajs");

exports.kafka = new Kafka({
  clientId: "my-app",
  brokers: ["<PRIVATE_IP>:9092"],
});

```
`admin.js`
```js
const { kafka } = require("./client");

async function init() {
  const admin = kafka.admin();
  console.log("Admin connecting...");
  admin.connect();
  console.log("Adming Connection Success...");

  console.log("Creating Topic [rider-updates]");
  await admin.createTopics({
    topics: [
      {
        topic: "rider-updates",
        numPartitions: 2,
      },
    ],
  });
  console.log("Topic Created Success [rider-updates]");

  console.log("Disconnecting Admin..");
  await admin.disconnect();
}

init();
```
`producer.js`
```js
const { kafka } = require("./client");
const readline = require("readline");

const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout,
});

async function init() {
  const producer = kafka.producer();

  console.log("Connecting Producer");
  await producer.connect();
  console.log("Producer Connected Successfully");

  rl.setPrompt("> ");
  rl.prompt();

  rl.on("line", async function (line) {
    const [riderName, location] = line.split(" ");
    await producer.send({
      topic: "rider-updates",
      messages: [
        {
          partition: location.toLowerCase() === "north" ? 0 : 1,
          key: "location-update",
          value: JSON.stringify({ name: riderName, location }),
        },
      ],
    });
  }).on("close", async () => {
    await producer.disconnect();
  });
}

init();
```
`consumer.js`
```js
const { kafka } = require("./client");
const group = process.argv[2];

async function init() {
  const consumer = kafka.consumer({ groupId: group });
  await consumer.connect();

  await consumer.subscribe({ topics: ["rider-updates"], fromBeginning: true });

  await consumer.run({
    eachMessage: async ({ topic, partition, message, heartbeat, pause }) => {
      console.log(
        `${group}: [${topic}]: PART:${partition}:`,
        message.value.toString()
      );
    },
  });
}

init();
```
## Running Locally
- Run Multiple Consumers
```bash
node consumer.js <GROUP_NAME>
```
- Create Producer
```bash
node producer.js
```
```bash
> tony south
> tony north
```








## ✅ How is Data Inserted into the DB?

### 1️⃣ Buffering with Batching (Recommended)
Instead of inserting data row by row, the consumer batches multiple Kafka messages and inserts them into the database at regular intervals (e.g., every 1 second or after collecting 1,000 records).  
This minimizes the number of expensive DB writes.

### 📌 Implementation (Example using PostgreSQL & Kafka Consumer)

```javascript
const { Kafka } = require("kafkajs");
const { Pool } = require("pg");

const kafka = new Kafka({ clientId: "zomato-consumer", brokers: ["localhost:9092"] });
const consumer = kafka.consumer({ groupId: "location-consumers" });

const pool = new Pool({ connectionString: "postgres://user:password@localhost:5432/zomato" });

let buffer = []; // Stores messages temporarily
const BATCH_SIZE = 1000; // Adjust based on DB performance

const insertBatchToDB = async () => {
  if (buffer.length === 0) return;

  const values = buffer.map(l => `('${l.driver_id}', ${l.latitude}, ${l.longitude}, '${l.timestamp}')`).join(",");
  const query = `INSERT INTO driver_locations (driver_id, latitude, longitude, timestamp) VALUES ${values}`;
  
  try {
    await pool.query(query);
    console.log(`✅ Inserted ${buffer.length} records into DB`);
    buffer = []; // Clear buffer after inserting
  } catch (error) {
    console.error("❌ Error inserting into DB:", error);
  }
};

const runConsumer = async () => {
  await consumer.connect();
  await consumer.subscribe({ topic: "driver_location_updates", fromBeginning: false });

  await consumer.run({
    eachMessage: async ({ message }) => {
      const location = JSON.parse(message.value.toString());
      buffer.push(location);

      if (buffer.length >= BATCH_SIZE) {
        await insertBatchToDB();
      }
    },
  });

  setInterval(insertBatchToDB, 1000); // Inserts any remaining data every second
};

runConsumer();
```
