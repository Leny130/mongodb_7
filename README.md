# mongodb_7
В докере поднял percona
```
name: mongodb-pmm-monitoring

services:
  mongodb:
    image: mongo:6.0
    container_name: mongodb
    restart: unless-stopped
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password123
    volumes:
      - mongodb_data:/data/db
    command: [
      "--bind_ip_all",
      "--auth",
      "--slowms=100"
    ]
    networks:
      - monitoring-net

  pmm-server:
    image: percona/pmm-server:2
    container_name: pmm-server
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    environment:
      - ENABLE_DBAAS=0
    volumes:
      - pmm_data:/srv
    networks:
      - monitoring-net

volumes:
  mongodb_data:
    driver: local
  pmm_data:
    driver: local
```
клиент
```
name: mongodb-pmm-client

services:
  pmm-client:
    image: percona/pmm-client:2
    container_name: pmm-client
    restart: unless-stopped
    cap_add:
      - ALL
    privileged: true
    volumes:
      - /proc:/proc:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      - PMM_AGENT_SERVER_ADDRESS=pmm-server
      - PMM_AGENT_SERVER_USERNAME=admin
      - PMM_AGENT_SERVER_PASSWORD=admin
      - PMM_AGENT_SETUP=1
      - PMM_AGENT_SETUP_METRICS_MODE=push
      - PMM_AGENT_SETUP_NODE_NAME=mongodb-docker-node
    networks:
      - monitoring-net

networks:
  monitoring-net:
    name: mongodb-pmm-monitoring_monitoring-net
    external: true
```

<img width="2103" height="947" alt="Снимок экрана 2025-10-23 в 21 35 44" src="https://github.com/user-attachments/assets/25b9529c-7f12-47f7-a588-6f81b5098545" />

тест
```
const { MongoClient } = require('mongodb');

async function generateLoad() {
    const uri = "mongodb://admin:password123@mongodb:27017";
    const client = new MongoClient(uri);

    try {
        await client.connect();
        const db = client.db('load_test');
        
        console.log("Starting load test at:", new Date());

        // Создание коллекции
        try {
            await db.createCollection('test_data');
        } catch (e) {
            // Коллекция уже существует
        }
        
        // Массовая вставка
        const batchSize = 100;
        for (let batch = 0; batch < 10; batch++) {
            const operations = [];
            for (let i = 0; i < batchSize; i++) {
                operations.push({
                    insertOne: {
                        document: {
                            timestamp: new Date(),
                            data: "Sample data " + Math.random(),
                            index: batch * batchSize + i,
                            metadata: {
                                type: "load_test",
                                iteration: batch
                            }
                        }
                    }
                });
            }
            await db.collection('test_data').bulkWrite(operations);
            console.log(`Inserted batch ${batch + 1}/10`);
        }
        
        // Чтение данных
        for (let i = 0; i < 50; i++) {
            await db.collection('test_data').find({ index: { $lt: 100 } }).toArray();
        }
        console.log("Read operations completed");
        
        // Агрегации
        for (let i = 0; i < 20; i++) {
            await db.collection('test_data').aggregate([
                { $group: { _id: "$metadata.type", count: { $sum: 1 } } }
            ]).toArray();
        }
        console.log("Aggregation operations completed");
        
        console.log("Load test completed at:", new Date());
        
    } catch (error) {
        console.error("Error during load test:", error);
    } finally {
        await client.close();
    }
}

generateLoad().catch(console.error);
```
<img width="860" height="653" alt="Снимок экрана 2025-10-23 в 21 41 37" src="https://github.com/user-attachments/assets/8fa0a580-d493-4866-8ba4-70517a8c8420" />


