1. Payment metrics
 src/payment-metrics.js
const client = require('prom-client');

const paymentTotal = new client.Counter({
  name:       'payment_total',
  labelNames: ['status', 'currency']
});

const paymentAmount = new client.Histogram({
  name:       'payment_amount',
  labelNames: ['currency'],
  buckets:    [100, 500, 1000, 5000, 10000, 50000]
});

const paymentDuration = new client.Histogram({
  name:       'payment_processing_seconds',
  labelNames: ['type'],
  buckets:    [0.1, 0.5, 1, 2, 5, 10]
});

const revenueTotal = new client.Gauge({
  name: 'revenue_total_inr',
  async collect() {
    const { rows } = await db.query(
    SELECT COALESCE(SUM(amount),0) AS total
       FROM orders WHERE status='paid'
       AND paid_at > NOW() - INTERVAL '24 hours'
    );
    this.set(parseInt(rows[0].total));
  }
});

const failedPayments = new client.Gauge({
  name: 'payments_failed_24h',
  async collect() {
    const { rows } = await db.query(
      SELECT COUNT(*) FROM orders WHERE status='failed'
       AND created_at > NOW() - INTERVAL '24 hours'
    );
    this.set(parseInt(rows[0].count));
  }
});

const activeSubscriptions = new client.Gauge({
  name: 'subscriptions_active_total',
  async collect() {
    const { rows } = await db.query(
      SELECT COUNT(*) FROM subscriptions WHERE status='active'
    );
    this.set(parseInt(rows[0].count));
  }
});

module.exports = {
  paymentTotal,
  paymentAmount,
  paymentDuration,
  revenueTotal,
  failedPayments,
  activeSubscriptions
};

// src/handlers.js
const {
  paymentTotal,
  paymentAmount,
  paymentDuration
} = require('./payment-metrics');

const handlePaymentSuccess = async (intent) => {
  const end = paymentDuration.startTimer({ type: 'payment_intent' });

  const { rows } = await db.query(
    'SELECT status, amount, currency FROM orders WHERE payment_intent_id=$1',
    [intent.id]
  );

  if (!rows.length)              return { error: 'order not found' };
  if (rows[0].status === 'paid') return { skipped: 'already paid' };

  await db.query(
    'UPDATE orders SET status=$1, paid_at=NOW() WHERE payment_intent_id=$2',
    ['paid', intent.id]
  );

  paymentTotal.inc({ status: 'success', currency: rows[0].currency });
  paymentAmount.observe({ currency: rows[0].currency }, rows[0].amount);
  end();

  return { updated: true, status: 'paid' };
};

const handlePaymentFailed = async (intent) => {
  const { rows } = await db.query(
    'SELECT status, currency FROM orders WHERE payment_intent_id=$1',
    [intent.id]
  );

  if (!rows.length)               return { error: 'order not found' };
  if (rows[0].status === 'failed') return { skipped: 'already failed' };

  await db.query(
    'UPDATE orders SET status=$1 WHERE payment_intent_id=$2',
    ['failed', intent.id]
  );

  paymentTotal.inc({ status: 'failed', currency: rows[0].currency });
  return { updated: true, status: 'failed' };
};

2. Grafana dashboard
{
  "title": "Payment Monitoring",
  "panels": [
    {
      "title": "Revenue (24h)",
      "type": "stat",
      "targets": [{ "expr": "revenue_total_inr" }]
    },
    {
      "title": "Payment Success Rate",
      "type": "stat",
      "targets": [{
        "expr": "rate(payment_total{status='success'}[5m]) / rate(payment_total[5m]) * 100"
      }]
    },
    {
      "title": "Payments/min",
      "type": "graph",
      "targets": [{
        "expr": "rate(payment_total[1m]) * 60"
      }]
    },
    {
      "title": "Failed Payments (24h)",
      "type": "stat",
      "targets": [{ "expr": "payments_failed_24h" }]
    },
    {
      "title": "Payment Amount Distribution",
      "type": "heatmap",
      "targets": [{
        "expr": "rate(payment_amount_bucket[5m])"
      }]
    },
    {
      "title": "Processing Latency p95",
      "type": "graph",
      "targets": [{
        "expr": "histogram_quantile(0.95, rate(payment_processing_seconds_bucket[5m]))"
      }]
    },
    {
      "title": "Active Subscriptions",
      "type": "stat",
      "targets": [{ "expr": "subscriptions_active_total" }]
    },
    {
      "title": "Webhook Queue Depth",
      "type": "graph",
      "targets": [{ "expr": "webhook_queue_depth" }]
    }
  ]
}

3. Alerts
# k8s/production/payment-monitoring-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: payment-monitoring
  namespace: production
spec:
  groups:
    - name: payment-monitoring
      rules:
        - alert: PaymentSuccessRateLow
          expr: |
            rate(payment_total{status="success"}[5m]) /
            rate(payment_total[5m]) < 0.95
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Payment success rate below 95%"
        - alert: NoPaymentsReceived
          expr: rate(payment_total[15m]) == 0
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "No payments received in 15 minutes"
        - alert: HighFailedPayments
          expr: payments_failed_24h > 20
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "More than 20 failed payments in 24h"
        - alert: RevenueDropped
          expr: |
            revenue_total_inr
            < (revenue_total_inr offset 1h) * 0.5
          for: 10m
          labels:
            severity: critical
          annotations:
            summary: "Revenue dropped more than 50% vs 1h ago"
        - alert: SubscriptionsDeclining
          expr: |
            subscriptions_active_total
            < (subscriptions_active_total offset 1h) * 0.9
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Active subscriptions dropped 10% in 1h"
        - alert: SlowPaymentProcessing
          expr: |
            histogram_quantile(0.95,
              rate(payment_processing_seconds_bucket[5m])) > 10
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Payment p95 processing > 10s"

  kubectl apply -f k8s/production/payment-monitoring-alerts.yaml

  4. Demo
echo "=== trigger payments ==="
stripe trigger payment_intent.succeeded
stripe trigger payment_intent.payment_failed
stripe trigger customer.subscription.deleted

echo "=== metrics ==="
curl -s https://prod.placemux.com/metrics | grep -E "payment_|revenue_|subscription_"

echo "=== db summary ==="
kubectl exec -it <db-pod> -n production -- psql -U placemux -c \
  "SELECT status, COUNT(*), SUM(amount)
   FROM orders GROUP BY status;"

kubectl exec -it <db-pod> -n production -- psql -U placemux -c \
  "SELECT COUNT(*) FROM subscriptions WHERE status='active';"

echo "=== simulate low success rate ==="
kubectl exec -it <db-pod> -n production -- psql -U placemux -c \
  "UPDATE orders SET status='failed'
   WHERE created_at > NOW() - INTERVAL '1 hour'
   AND status='pending';"

sleep 30
curl -s https://prod.placemux.com/metrics | grep payment_total

Expected output:

success rate      >95%
revenue_total     sum of paid orders
failed_24h        count of failed orders
subscriptions     active count
alert             PaymentSuccessRateLow fires on simulation
dashboard         all panels populated in Grafana

<img width="1674" height="940" alt="ss1002" src="https://github.com/user-attachments/assets/af7d5c92-6ff5-4e2e-af83-6f236936937b" />
