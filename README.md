@PostMapping("/callback/{refId}")
public ResponseEntity<String> receiveCallback(@PathVariable String refId, @RequestBody Map<String, String> payload) {
    String status = payload.getOrDefault("status", "FAILURE");
    paymentService.completeTransaction(refId, status);
    return ResponseEntity.ok("Callback received for " + refId);
}

public void completeTransaction(String refId, String status) {
    SseEmitter emitter = emitters.get(refId);
    if (emitter != null) {
        try {
            emitter.send(status);
            emitter.complete();
        } catch (IOException e) {
            emitter.completeWithError(e);
        }
    }
}

public String createTransaction() {
    String refId = UUID.randomUUID().toString();
    SseEmitter emitter = new SseEmitter(0L);
    emitters.put(refId, emitter);
    return refId;
}

<button onclick="makePayment()">Start Payment</button>
<button onclick="simulateCallback()">Simulate NBBL Callback</button>
<p id="status"></p>

<script>
  let currentRefId = null;

  function makePayment() {
    fetch("/pay", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ amount: 100 })
    })
    .then(res => res.json())
    .then(({ refId, status, errorMessage }) => {
      if (status === "INITIATED") {
        currentRefId = refId;
        const source = new EventSource(`/pay/status/${refId}`);
        source.onmessage = e => {
          document.getElementById("status").innerText = e.data;
          if (["SUCCESS", "FAILURE"].includes(e.data)) source.close();
        };
      } else {
        alert("Error: " + errorMessage);
      }
    });
  }

  function simulateCallback() {
    if (!currentRefId) {
      alert("Please initiate a payment first.");
      return;
    }

    fetch(`/pay/callback/${currentRefId}`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ status: "SUCCESS" }) // or "FAILURE"
    })
    .then(res => res.text())
    .then(msg => alert(msg));
  }
</script>
