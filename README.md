# CST8917 Lab4: Real-Time Trip Event Analysis

## Project Overview
This lab implements a **Real-Time Trip Monitoring for Taxi Dispatch System** using Azure services. It ingests trip data from an Event Hub, processes it with an Azure Function, and routes alerts to Microsoft Teams via Logic Apps and Adaptive Cards. The system identifies whether the trip is interesting or not, or suspicious trip patterns to help operations teams take action immediately.

## Project Architecture
- Azure Event Hub for ingest taxi events
- Azure function for analyzing and monitor taxi events
- Azure logic apps for post adaptive card to teams

## Azure Function
We create azure function to analyze and monitor taxi events ingest by azure event hub.
it use HTTO Trigger called by logic app using POST requestion with trip events.
it contains vendorID, tripDistance, passengerCount, paymentType.
it also analyze the conditions isInteresting: true with insight tags (if flagged), and isInteresting: false (if no issues found)
<img width="1906" height="906" alt="image" src="https://github.com/user-attachments/assets/baa7033d-a5e2-4c89-a029-607818ccd0e1" />

```python
import azure.functions as func
import logging
import json

app = func.FunctionApp(http_auth_level=func.AuthLevel.ANONYMOUS)

@app.route(route="")
def analyze_trip(req: func.HttpRequest) -> func.HttpResponse:
    try:
        input_data = req.get_json()
        trips = input_data if isinstance(input_data, list) else [input_data]

        results = []

        for record in trips:
            trip = record.get("ContentData", {})  # ✅ Extract inner trip data

            vendor = trip.get("vendorID")
            distance = float(trip.get("tripDistance", 0))
            passenger_count = int(trip.get("passengerCount", 0))
            payment = str(trip.get("paymentType"))  # Cast to string to match logic

            insights = []

            if distance > 10:
                insights.append("LongTrip")
            if passenger_count > 4:
                insights.append("GroupRide")
            if payment == "2":
                insights.append("CashPayment")
            if payment == "2" and distance < 1:
                insights.append("SuspiciousVendorActivity")

            results.append({
                "vendorID": vendor,
                "tripDistance": distance,
                "passengerCount": passenger_count,
                "paymentType": payment,
                "insights": insights,
                "isInteresting": bool(insights),
                "summary": f"{len(insights)} flags: {', '.join(insights)}" if insights else "Trip normal"
            })

        return func.HttpResponse(
            body=json.dumps(results),
            status_code=200,
            mimetype="application/json"
        )

    except Exception as e:
        logging.error(f"Error processing trip data: {e}")
        return func.HttpResponse(f"Error: {str(e)}", status_code=400)
```
it records four conditions
- distance > 10, for long trip.
- passenger number > 4, for group ride.
- payment type == 2, for cash payment.
- payment type == 2 and distance < 1, for suspicious vendor activity.

## Azure Event Hub
Azure event hub will ingest taxi events to azure function for analyzing.
example input:
```json
{
    "vendorID": "3",
    "tpepPickupDateTime": 1528119858000,
    "tpepDropoffDateTime": 1528121148000,
    "passengerCount": 2,
    "tripDistance": 5.66,
    "puLocationId": "186",
    "doLocationId": "230",
    "startLon": null,
    "startLat": null,
    "endLon": null,
    "endLat": null,
    "rateCodeId": 1,
    "storeAndFwdFlag": "N",
    "paymentType": 1,
    "fareAmount": 13.5,
    "extra": 0,
    "mtaTax": 0.5,
    "improvementSurcharge": "0.3",
    "tipAmount": 2.86,
    "tollsAmount": 0,
    "totalAmount": 17.16
}
```
example output:
```json
{
  "statusCode": 200,
  "headers": {
    "Date": "Fri, 01 Aug 2025 08:47:22 GMT",
    "Server": "Kestrel",
    "Transfer-Encoding": "chunked",
    "Content-Type": "application/json",
    "Content-Length": "148"
  },
  "body": [
    {
      "vendorID": "3",
      "tripDistance": 3.41,
      "passengerCount": 3,
      "paymentType": "1",
      "insights": [],
      "isInteresting": false,
      "summary": "Trip normal"
    }
  ]
}
```
indicate **Trip Analyzed - No Issues**

## Azure Logic App
Azure logic app will process the taxi event ingest through azure function analysis, and send adpative cards to teams base on coditions.
<img width="1903" height="909" alt="image" src="https://github.com/user-attachments/assets/6ca6195e-8fc6-40b9-999d-d167835ef259" />

<img width="1896" height="909" alt="image" src="https://github.com/user-attachments/assets/96a08a4e-b81a-498c-83a3-dcd4d1f7a3f5" />
It sends **Trip Aanalyzed - No Issues** when the condition of is intesting is false by sending json file of

```json
  {
  "type": "AdaptiveCard",
  "body": [
    {
      "type": "TextBlock",
      "text": "✅ Trip Analyzed - No Issues",
      "weight": "Bolder",
      "size": "Large",
      "color": "Good"
    },
    {
      "type": "FactSet",
      "facts": [
        { "title": "Vendor", "value": "@{items('For_each')?['vendorID']}" },
        { "title": "Distance (mi)", "value": "@{items('For_each')?['tripDistance']}" },
        { "title": "Passengers", "value": "@{items('For_each')?['passengerCount']}" },
        { "title": "Payment", "value": "@{items('For_each')?['paymentType']}" },
        { "title": "Summary", "value": "@{items('For_each')?['summary']}" }
      ]
    }
  ],
  "actions": [],
  "version": "1.2"
}
```

<img width="1898" height="878" alt="image" src="https://github.com/user-attachments/assets/4ee4e53f-cd0d-4eb5-b474-60985f6cbe2a" />
<img width="456" height="201" alt="image" src="https://github.com/user-attachments/assets/0e47e3a2-4e90-41fe-ab8a-beb3df9bc926" />


It sends **Ineresting Trip Detected** when the condition of is intesting is true and contain suspiciousvenderactivity is false by sending json file of

```json
  {
  "type": "AdaptiveCard",
  "body": [
    {
      "type": "TextBlock",
      "text": "🚨 Interesting Trip Detected",
      "weight": "Bolder",
      "size": "Large",
      "color": "Attention"
    },
    {
      "type": "FactSet",
      "facts": [
        { "title": "Vendor", "value": "@{items('For_each')?['vendorID']}" },
        { "title": "Distance (mi)", "value": "@{items('For_each')?['tripDistance']}" },
        { "title": "Passengers", "value": "@{items('For_each')?['passengerCount']}" },
        { "title": "Payment", "value": "@{items('For_each')?['paymentType']}" },
        { "title": "Insights", "value": "@{join(items('For_each')?['insights'], ', ')}" }
      ]
    }
  ],
  "actions": [],
  "version": "1.2"
}

```
<img width="1902" height="907" alt="image" src="https://github.com/user-attachments/assets/34000753-255e-4118-936f-88a02394cb45" />
<img width="457" height="198" alt="image" src="https://github.com/user-attachments/assets/952b460e-b080-4334-bf7c-9b9ca085d0dc" />

It sends **Supicious Vendor Activity Detected** when the condition of is intesting is true and contain supicousvendoractvitiy is true by sending json file of

```json
  {
  "type": "AdaptiveCard",
  "body": [
    {
      "type": "TextBlock",
      "text": "⚠️ Suspicious Vendor Activity Detected",
      "weight": "Bolder",
      "size": "Large",
      "color": "Attention"
    },
    {
      "type": "FactSet",
      "facts": [
        { "title": "Vendor", "value": "@{items('For_each')?['vendorID']}" },
        { "title": "Distance (mi)", "value": "@{items('For_each')?['tripDistance']}" },
        { "title": "Passengers", "value": "@{items('For_each')?['passengerCount']}" },
        { "title": "Payment", "value": "@{items('For_each')?['paymentType']}" },
        { "title": "Insights", "value": "@{join(items('For_each')?['insights'], ', ')}" }
      ]
    }
  ],
  "actions": [],
  "version": "1.2"
}
```

<img width="1885" height="910" alt="image" src="https://github.com/user-attachments/assets/454af13e-3955-4073-83f2-3ca47d519872" />
<img width="458" height="197" alt="image" src="https://github.com/user-attachments/assets/e88987a1-377a-4a38-b95d-ad83c2a794e3" />

## Improvements
- Use Durable Functiosn for stateful orchestraction
- Add locations for record where taxi goes
- Add flagged data into sql to store card histories
- Add alert level for different warrant urgency

## Youtube video link
https://youtu.be/R8N1k8oiw_Q
