import com.razorpay.Order;
import com.razorpay.RazorpayClient;
import com.razorpay.RazorpayException;
import org.json.JSONObject;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/payment")
public class PaymentController {

    private final RazorpayClient razorpayClient;

    public PaymentController(@Value("${razorpay.key}") String keyId,
                             @Value("${razorpay.secret}") String keySecret) throws RazorpayException {
        this.razorpayClient = new RazorpayClient(keyId, keySecret);
    }

    @PostMapping("/create")
    public String createOrder(@RequestParam("amount") int amount) throws RazorpayException {
        JSONObject orderRequest = new JSONObject();
        orderRequest.put("amount", amount * 100); // Amount in paise
        orderRequest.put("currency", "INR");
        orderRequest.put("payment_capture", 1); // Auto-capture

        Order order = razorpayClient.orders.create(orderRequest);
        return order.get("id").toString();
    }

    @PostMapping("/verify")
    public String verifyPayment(@RequestParam("razorpay_order_id") String orderId,
                                 @RequestParam("razorpay_payment_id") String paymentId,
                                 @RequestParam("razorpay_signature") String signature) {
        // Implement signature verification logic here
        return "Payment verification successful";
    }
}
import com.razorpay.RazorpayClient;
import com.razorpay.RazorpayException;
import org.json.JSONObject;

public boolean verifyPayment(String orderId, String paymentId, String signature) {
    try {
        JSONObject paymentDetails = new JSONObject();
        paymentDetails.put("razorpay_order_id", orderId);
        paymentDetails.put("razorpay_payment_id", paymentId);
        paymentDetails.put("razorpay_signature", signature);

        // Use Razorpay's utility to verify the signature
        razorpayClient.utility.verifyPaymentSignature(paymentDetails);
        return true;
    } catch (RazorpayException e) {
        // Handle exception
        return false;
    }
}
@Entity
public class PaymentTransaction {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String orderId;
    private String paymentId;
    private String signature;
    private int amount;
    private String status; // CREATED, SUCCESS, FAILED
    private LocalDateTime createdAt;

    // Getters, Setters
}
@Repository
public interface PaymentTransactionRepository extends JpaRepository<PaymentTransaction, Long> {
    Optional<PaymentTransaction> findByOrderId(String orderId);
}
@Autowired
private PaymentTransactionRepository paymentRepo;

@PostMapping("/create")
public String createOrder(@RequestParam("amount") int amount) throws RazorpayException {
    JSONObject orderRequest = new JSONObject();
    orderRequest.put("amount", amount * 100);
    orderRequest.put("currency", "INR");
    orderRequest.put("payment_capture", 1);

    Order order = razorpayClient.orders.create(orderRequest);

    // Save to DB
    PaymentTransaction txn = new PaymentTransaction();
    txn.setOrderId(order.get("id"));
    txn.setAmount(amount);
    txn.setStatus("CREATED");
    txn.setCreatedAt(LocalDateTime.now());
    paymentRepo.save(txn);

    return order.get("id");
}
