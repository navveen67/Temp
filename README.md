package com.naveen.paymentsse.dto;

import jakarta.validation.constraints.Min;
import lombok.Data;

@Data
public class PaymentRequest {
    @Min(value = 1, message = "Amount must be greater than zero")
    private int amount;
}


package com.naveen.paymentsse.dto;

import lombok.AllArgsConstructor;
import lombok.Data;

@Data
@AllArgsConstructor
public class PaymentResponse {
    private String refId;
    private String status;
    private String errorMessage;
}

package com.naveen.paymentsse.service;

import org.springframework.stereotype.Service;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

import java.util.Map;
import java.util.Random;
import java.util.UUID;
import java.util.concurrent.*;

@Service
public class PaymentService {

    private final Map<String, SseEmitter> emitters = new ConcurrentHashMap<>();

    public String createTransaction() {
        String refId = UUID.randomUUID().toString();
        SseEmitter emitter = new SseEmitter(0L);
        emitters.put(refId, emitter);
        simulateProcessing(refId);
        return refId;
    }

    public SseEmitter getStatusStream(String refId) {
        return emitters.get(refId);
    }

    private void simulateProcessing(String refId) {
        CompletableFuture.runAsync(() -> {
            try {
                SseEmitter emitter = emitters.get(refId);
                emitter.send("IN_PROGRESS");
                Thread.sleep(3000); // Simulated delay
                boolean isSuccess = new Random().nextBoolean();
                emitter.send(isSuccess ? "SUCCESS" : "FAILURE");
                emitter.complete();
            } catch (Exception e) {
                emitters.get(refId).completeWithError(e);
            }
        });
    }
}

package com.naveen.paymentsse.controller;

import com.naveen.paymentsse.dto.PaymentRequest;
import com.naveen.paymentsse.dto.PaymentResponse;
import com.naveen.paymentsse.service.PaymentService;
import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

@RestController
@RequestMapping("/pay")
public class PaymentController {

    private final PaymentService paymentService;

    public PaymentController(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    @PostMapping
    public ResponseEntity<PaymentResponse> initiate(@RequestBody @Valid PaymentRequest request) {
        String refId = paymentService.createTransaction();
        return ResponseEntity.ok(new PaymentResponse(refId, "INITIATED", null));
    }

    @GetMapping(value = "/status/{refId}", produces = "text/event-stream")
    public SseEmitter streamStatus(@PathVariable String refId) {
        return paymentService.getStatusStream(refId);
    }
}



<!DOCTYPE html>
<html>
<head>
  <title>Payment SSE</title>
</head>
<body>
  <h2>Make a Payment</h2>
  <button onclick="makePayment()">Start Payment</button>
  <p id="status"></p>

  <script>
    function makePayment() {
      fetch("/pay", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ amount: 100 })
      })
      .then(res => res.json())
      .then(({ refId, status, errorMessage }) => {
        if (status === "INITIATED") {
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
  </script>
</body>
</html>


@Bean
public WebMvcConfigurer corsConfigurer() {
    return registry -> registry.addMapping("/**").allowedOrigins("*");
}
