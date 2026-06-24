# Transaction Monitoring & Fraud Detection POC

A data engineering system built on **Microsoft Fabric** and **Azure**, implementing real-time streaming ingestion, Medallion Architecture (Bronze → Silver → Gold), SCD Type 2 dimensions, watermark-based incremental loading, and a composable 5-rule fraud detection engine — serving a Power BI dashboard with branch-level Row-Level Security across 500 accounts and 6,000+ transactions.

🌐 **Full documentation, architecture, and demo:** [poc.tusharkolte.com](https://poc.tusharkolte.com)

---

## Key Implementation Snippets

### 1. PII Masking — Silver Layer

Raw PII columns (`email`, `phone`, `pan`, `date_of_birth`) are **dropped**, not just masked. Structural impossibility of leakage to Gold.

```python
def mask_email(c): 
    return expr(f"concat(substring({c},1,1),'***@',substring({c},instr({c},'@')+1,length({c})))")

def mask_phone(c): 
    return expr(f"concat(repeat('X',length({c})-4),substring({c},length({c})-3,4))")

def mask_pan(c):   
    return expr(f"concat('XXXXX',substring({c},6,5))")

def age_bucket(dob):
    today = date.today().isoformat()
    return (
        when(datediff(lit(today), col(dob)) / 365.25 < 26, "18-25")
        .when(datediff(lit(today), col(dob)) / 365.25 < 36, "26-35")
        .when(datediff(lit(today), col(dob)) / 365.25 < 51, "36-50")
        .otherwise("51+")
    )

df_silver = (
    df_raw
    .withColumn("email_masked", mask_email("email"))
    .withColumn("phone_masked", mask_phone("phone"))
    .withColumn("pan_masked",   mask_pan("pan"))
    .withColumn("age_bucket",   age_bucket("date_of_birth"))
    .drop("email", "phone", "pan", "date_of_birth")  # source columns removed
)
```

---

### 2. Watermark-Based Incremental Loading

Central `_pipeline_watermarks` Delta table tracks last-processed state per layer. Epoch zero forces full load on first run.

```python
# Read watermark
last_watermark = (
    spark.table("_pipeline_watermarks")
         .filter("layer_name = 'silver_transactions'")
         .select("watermark_ts")
         .collect()[0]["watermark_ts"]
)

# Process only new rows
df_new = spark.table("raw_historical_transactions").filter(
    col("ingestion_timestamp") > last_watermark
)

# ... transform and write ...

# Update watermark to max ingestion_timestamp of this batch
max_ts = df_new.agg(F.max("ingestion_timestamp")).collect()[0][0]
spark.sql(f"""
    UPDATE _pipeline_watermarks
    SET watermark_ts = '{max_ts}', rows_last_run = {new_count},
        last_run_status = 'SUCCESS', last_run_at = current_timestamp()
    WHERE layer_name = 'silver_transactions'
""")
```

> **Isolation fix:** Set `delta.isolationLevel = WriteSerializable` on `_pipeline_watermarks` to prevent `ConcurrentAppendException` when parallel pipeline activities update different rows simultaneously.

---


### 3. Velocity Fraud Rule — Window Function

Counts transactions per account in a rolling 1-hour window. `rangeBetween` requires `unix_timestamp` (integer seconds) not raw datetime.

```python
window_velocity = (
    Window
    .partitionBy("account_id")
    .orderBy(F.unix_timestamp("transaction_timestamp"))
    .rangeBetween(-3600, 0)        # -3600 seconds = exactly 1 hour back
)

df = (
    df_active
    .withColumn("txn_count_in_window",
        F.count("transaction_id").over(window_velocity).cast(IntegerType()))
    .withColumn("flag_velocity",
        (col("txn_count_in_window") > 3).cast(BooleanType()))
)
```

---

### 4. Azure Functions — Autonomous Live Producer

Timer-triggered function sends 10–20 events to Event Hubs every 5 minutes with periodic fraud injections. No laptop required.

```python
@app.schedule(schedule="0 */5 * * * *", arg_name="timer",
              run_on_startup=False, use_monitor=False)
def txn_producer(timer: func.TimerRequest) -> None:
    conn_str = os.environ["EVENTHUB_CONN_STR"]
    events   = [make_event(f"{i:03d}") for i in range(random.randint(10, 18))]

    minute = datetime.now(timezone.utc).minute
    if minute % 25 < 5:                            # HIGH_VALUE every ~25 min
        events.append(make_event("HV0", amount=round(random.uniform(6000, 80000), 2)))
    if minute % 50 < 5:                            # VELOCITY burst every ~50 min
        burst = random.choice(ACCOUNT_IDS)
        for j in range(4):
            events.append(make_event(f"VB{j}", account_id=burst))

    producer = EventHubProducerClient.from_connection_string(conn_str=conn_str,
                                                              eventhub_name="txn-events")
    with producer:
        batch = producer.create_batch()
        for e in events:
            batch.add(EventData(json.dumps(e)))
        producer.send_batch(batch)
```

---

