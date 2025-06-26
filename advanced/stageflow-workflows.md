# StageFlow Workflows - Real-World Examples

This guide demonstrates how to build complex, multi-stage workflows using StageFlow in both .NET (Mmate.StageFlow) and Go implementations with practical scenarios.

## Table of Contents
- [E-commerce Order Fulfillment](#e-commerce-order-fulfillment)
- [Loan Application Processing](#loan-application-processing)
- [Document Processing Pipeline](#document-processing-pipeline)
- [User Onboarding Workflow](#user-onboarding-workflow)
- [Expense Approval Workflow](#expense-approval-workflow)
- [Manufacturing Process Control](#manufacturing-process-control)

## E-commerce Order Fulfillment

### Scenario: Complete Order Processing Workflow

Process orders through validation, payment, inventory, shipping, and notification stages.

### 1. Define Workflow State

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class OrderFulfillmentState
{
    public string OrderId { get; set; }
    public string CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
    public decimal TotalAmount { get; set; }
    public Address ShippingAddress { get; set; }
    public Address BillingAddress { get; set; }
    
    // Stage results
    public bool IsValidated { get; set; }
    public string ValidationMessage { get; set; }
    
    public bool InventoryReserved { get; set; }
    public Dictionary<string, int> ReservedItems { get; set; }
    
    public bool PaymentProcessed { get; set; }
    public string PaymentTransactionId { get; set; }
    public string PaymentMethod { get; set; }
    
    public bool ShippingArranged { get; set; }
    public string TrackingNumber { get; set; }
    public string Carrier { get; set; }
    public DateTime EstimatedDelivery { get; set; }
    
    public List<string> ProcessingErrors { get; set; } = new List<string>();
}

```

</td>
<td>

```go
type OrderFulfillmentState struct {
    OrderID         string              `json:"order_id"`
    CustomerID      string              `json:"customer_id"`
    Items           []OrderItem         `json:"items"`
    TotalAmount     decimal.Decimal     `json:"total_amount"`
    ShippingAddress Address             `json:"shipping_address"`
    BillingAddress  Address             `json:"billing_address"`
    
    // Stage results
    IsValidated       bool                `json:"is_validated"`
    ValidationMessage string              `json:"validation_message"`
    
    InventoryReserved bool                `json:"inventory_reserved"`
    ReservedItems     map[string]int      `json:"reserved_items"`
    
    PaymentProcessed      bool      `json:"payment_processed"`
    PaymentTransactionID  string    `json:"payment_transaction_id"`
    PaymentMethod         string    `json:"payment_method"`
    
    ShippingArranged   bool      `json:"shipping_arranged"`
    TrackingNumber     string    `json:"tracking_number"`
    Carrier            string    `json:"carrier"`
    EstimatedDelivery  time.Time `json:"estimated_delivery"`
    
    ProcessingErrors []string `json:"processing_errors"`
}
```

</td>
</tr>
</table>

### 2. Setup the Workflow

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class OrderFulfillmentWorkflow
{
    private readonly IFlowEndpointFactory _factory;
    private readonly ILogger<OrderFulfillmentWorkflow> _logger;
    
    public void Configure()
    {
        var workflow = _factory.Staged<OrderRequest, OrderFulfillmentState>("OrderFulfillment");
        
        // Stage 1: Validate Order
        workflow.Stage<OrderRequest>(async (context, state, request) =>
        {
            _logger.LogInformation("Stage 1: Validating order for customer {CustomerId}", 
                request.CustomerId);
            
            // Initialize state
            state.OrderId = Guid.NewGuid().ToString();
            state.CustomerId = request.CustomerId;
            state.Items = request.Items;
            state.ShippingAddress = request.ShippingAddress;
            state.BillingAddress = request.BillingAddress;
            state.TotalAmount = request.Items.Sum(i => i.Price * i.Quantity);
            
            // Validate order
            var validationRequest = new ValidateOrderRequest
            {
                OrderId = state.OrderId,
                CustomerId = state.CustomerId,
                Items = state.Items,
                TotalAmount = state.TotalAmount
            };
            
            await context.Request("OrderValidationService", validationRequest);
        });
        
        // Stage 2: Check and Reserve Inventory
        workflow.Stage<ValidateOrderResponse>(async (context, state, response) =>
        {
            _logger.LogInformation("Stage 2: Checking inventory for order {OrderId}", 
                state.OrderId);
            
            state.IsValidated = response.IsValid;
            state.ValidationMessage = response.Message;
            
            if (!response.IsValid)
            {
                state.ProcessingErrors.Add($"Validation failed: {response.Message}");
                return; // Skip to final stage
            }
            
            // Check inventory for all items
            var inventoryRequest = new ReserveInventoryRequest
            {
                OrderId = state.OrderId,
                Items = state.Items.Select(i => new InventoryItem
                {
                    ProductId = i.ProductId,
                    Quantity = i.Quantity,
                    WarehouseId = DetermineWarehouse(state.ShippingAddress)
                }).ToList()
            };
            
            await context.Request("InventoryService", inventoryRequest);
        });
        
        // Stage 3: Process Payment
        workflow.Stage<ReserveInventoryResponse>(async (context, state, response) =>
        {
            _logger.LogInformation("Stage 3: Processing payment for order {OrderId}", 
                state.OrderId);
            
            state.InventoryReserved = response.Success;
            state.ReservedItems = response.ReservedItems;
            
            if (!response.Success)
            {
                state.ProcessingErrors.Add($"Inventory unavailable: {response.Message}");
                
                // Release any partial reservations
                if (response.ReservedItems?.Any() == true)
                {
                    await context.Send("InventoryService", new ReleaseInventoryCommand
                    {
                        OrderId = state.OrderId,
                        Items = response.ReservedItems
                    });
                }
                return;
            }
            
            // Process payment
            var paymentRequest = new ProcessPaymentRequest
            {
                OrderId = state.OrderId,
                CustomerId = state.CustomerId,
                Amount = state.TotalAmount,
                BillingAddress = state.BillingAddress,
                SavePaymentMethod = true
            };
            
            await context.Request("PaymentService", paymentRequest);
        });
        
        // Stage 4: Arrange Shipping
        workflow.Stage<ProcessPaymentResponse>(async (context, state, response) =>
        {
            _logger.LogInformation("Stage 4: Arranging shipping for order {OrderId}", 
                state.OrderId);
            
            state.PaymentProcessed = response.Success;
            state.PaymentTransactionId = response.TransactionId;
            state.PaymentMethod = response.PaymentMethod;
            
            if (!response.Success)
            {
                state.ProcessingErrors.Add($"Payment failed: {response.Message}");
                
                // Release inventory reservation
                await context.Send("InventoryService", new ReleaseInventoryCommand
                {
                    OrderId = state.OrderId,
                    Items = state.ReservedItems
                });
                return;
            }
            
            // Create shipping request
            var shippingRequest = new CreateShipmentRequest
            {
                OrderId = state.OrderId,
                Items = state.Items,
                ShippingAddress = state.ShippingAddress,
                CustomerTier = await GetCustomerTier(state.CustomerId),
                PreferredCarrier = response.PaymentMethod == "Premium" ? "Express" : "Standard"
            };
            
            await context.Request("ShippingService", shippingRequest);
        });
        
        // Final Stage: Complete order and send notifications
        workflow.LastStage<CreateShipmentResponse, OrderFulfillmentResult>(
            async (context, state, response) =>
        {
            _logger.LogInformation("Final Stage: Completing order {OrderId}", state.OrderId);
            
            state.ShippingArranged = response.Success;
            if (response.Success)
            {
                state.TrackingNumber = response.TrackingNumber;
                state.Carrier = response.Carrier;
                state.EstimatedDelivery = response.EstimatedDelivery;
            }
            else
            {
                state.ProcessingErrors.Add($"Shipping failed: {response.Message}");
            }
            
            // Determine final status
            var success = state.IsValidated && 
                         state.InventoryReserved && 
                         state.PaymentProcessed && 
                         state.ShippingArranged;
            
            if (success)
            {
                // Send success notifications
                await context.Publish(new OrderCompletedEvent
                {
                    OrderId = state.OrderId,
                    CustomerId = state.CustomerId,
                    TotalAmount = state.TotalAmount,
                    TrackingNumber = state.TrackingNumber,
                    EstimatedDelivery = state.EstimatedDelivery
                });
                
                // Send confirmation email
                await context.Send("NotificationService", new SendOrderConfirmationCommand
                {
                    OrderId = state.OrderId,
                    CustomerId = state.CustomerId,
                    Email = await GetCustomerEmail(state.CustomerId),
                    TrackingNumber = state.TrackingNumber
                });
            }
            else
            {
                // Handle failure - refund if payment was processed
                if (state.PaymentProcessed)
                {
                    await context.Send("PaymentService", new RefundPaymentCommand
                    {
                        OrderId = state.OrderId,
                        TransactionId = state.PaymentTransactionId,
                        Amount = state.TotalAmount,
                        Reason = string.Join("; ", state.ProcessingErrors)
                    });
                }
                
                // Notify customer of failure
                await context.Publish(new OrderFailedEvent
                {
                    OrderId = state.OrderId,
                    CustomerId = state.CustomerId,
                    Errors = state.ProcessingErrors
                });
            }
            
            return new OrderFulfillmentResult
            {
                OrderId = state.OrderId,
                Success = success,
                Status = success ? "Completed" : "Failed",
                TrackingNumber = state.TrackingNumber,
                EstimatedDelivery = state.EstimatedDelivery,
                Errors = state.ProcessingErrors
            };
        });
    }
}
```

</td>
<td>

```go
type OrderFulfillmentWorkflow struct {
    dispatcher messaging.Dispatcher
    logger     *slog.Logger
}

func (w *OrderFulfillmentWorkflow) Configure() *stageflow.Workflow {
    workflow := stageflow.NewWorkflow[*OrderRequest, *OrderFulfillmentState]("OrderFulfillment")
    
    // Stage 1: Validate Order
    workflow.Stage(func(ctx context.Context, state *OrderFulfillmentState, request *OrderRequest) error {
        w.logger.Info("Stage 1: Validating order for customer", "customer_id", request.CustomerID)
        
        // Initialize state
        state.OrderID = uuid.New().String()
        state.CustomerID = request.CustomerID
        state.Items = request.Items
        state.ShippingAddress = request.ShippingAddress
        state.BillingAddress = request.BillingAddress
        
        var total decimal.Decimal
        for _, item := range request.Items {
            total = total.Add(item.Price.Mul(decimal.NewFromInt(int64(item.Quantity))))
        }
        state.TotalAmount = total
        
        // Validate order
        validationRequest := &ValidateOrderRequest{
            OrderID:     state.OrderID,
            CustomerID:  state.CustomerID,
            Items:       state.Items,
            TotalAmount: state.TotalAmount,
        }
        
        return w.dispatcher.Request(ctx, "OrderValidationService", validationRequest)
    })
    
    // Stage 2: Check and Reserve Inventory
    workflow.Stage(func(ctx context.Context, state *OrderFulfillmentState, response *ValidateOrderResponse) error {
        w.logger.Info("Stage 2: Checking inventory for order", "order_id", state.OrderID)
        
        state.IsValidated = response.IsValid
        state.ValidationMessage = response.Message
        
        if !response.IsValid {
            state.ProcessingErrors = append(state.ProcessingErrors, 
                fmt.Sprintf("Validation failed: %s", response.Message))
            return nil // Skip to final stage
        }
        
        // Check inventory for all items
        inventoryItems := make([]InventoryItem, len(state.Items))
        for i, item := range state.Items {
            inventoryItems[i] = InventoryItem{
                ProductID:   item.ProductID,
                Quantity:    item.Quantity,
                WarehouseID: w.determineWarehouse(state.ShippingAddress),
            }
        }
        
        inventoryRequest := &ReserveInventoryRequest{
            OrderID: state.OrderID,
            Items:   inventoryItems,
        }
        
        return w.dispatcher.Request(ctx, "InventoryService", inventoryRequest)
    })
    
    // Stage 3: Process Payment
    workflow.Stage(func(ctx context.Context, state *OrderFulfillmentState, response *ReserveInventoryResponse) error {
        w.logger.Info("Stage 3: Processing payment for order", "order_id", state.OrderID)
        
        state.InventoryReserved = response.Success
        state.ReservedItems = response.ReservedItems
        
        if !response.Success {
            state.ProcessingErrors = append(state.ProcessingErrors, 
                fmt.Sprintf("Inventory unavailable: %s", response.Message))
            
            // Release any partial reservations
            if len(response.ReservedItems) > 0 {
                releaseCmd := &ReleaseInventoryCommand{
                    OrderID: state.OrderID,
                    Items:   response.ReservedItems,
                }
                return w.dispatcher.Send(ctx, "InventoryService", releaseCmd)
            }
            return nil
        }
        
        // Process payment
        paymentRequest := &ProcessPaymentRequest{
            OrderID:           state.OrderID,
            CustomerID:        state.CustomerID,
            Amount:            state.TotalAmount,
            BillingAddress:    state.BillingAddress,
            SavePaymentMethod: true,
        }
        
        return w.dispatcher.Request(ctx, "PaymentService", paymentRequest)
    })
    
    // Stage 4: Arrange Shipping
    workflow.Stage(func(ctx context.Context, state *OrderFulfillmentState, response *ProcessPaymentResponse) error {
        w.logger.Info("Stage 4: Arranging shipping for order", "order_id", state.OrderID)
        
        state.PaymentProcessed = response.Success
        state.PaymentTransactionID = response.TransactionID
        state.PaymentMethod = response.PaymentMethod
        
        if !response.Success {
            state.ProcessingErrors = append(state.ProcessingErrors, 
                fmt.Sprintf("Payment failed: %s", response.Message))
            
            // Release inventory reservation
            releaseCmd := &ReleaseInventoryCommand{
                OrderID: state.OrderID,
                Items:   state.ReservedItems,
            }
            return w.dispatcher.Send(ctx, "InventoryService", releaseCmd)
        }
        
        // Determine preferred carrier
        preferredCarrier := "Standard"
        if response.PaymentMethod == "Premium" {
            preferredCarrier = "Express"
        }
        
        customerTier, err := w.getCustomerTier(ctx, state.CustomerID)
        if err != nil {
            return err
        }
        
        // Create shipping request
        shippingRequest := &CreateShipmentRequest{
            OrderID:          state.OrderID,
            Items:            state.Items,
            ShippingAddress:  state.ShippingAddress,
            CustomerTier:     customerTier,
            PreferredCarrier: preferredCarrier,
        }
        
        return w.dispatcher.Request(ctx, "ShippingService", shippingRequest)
    })
    
    // Final Stage: Complete order and send notifications
    workflow.LastStage(func(ctx context.Context, state *OrderFulfillmentState, response *CreateShipmentResponse) (*OrderFulfillmentResult, error) {
        w.logger.Info("Final Stage: Completing order", "order_id", state.OrderID)
        
        state.ShippingArranged = response.Success
        if response.Success {
            state.TrackingNumber = response.TrackingNumber
            state.Carrier = response.Carrier
            state.EstimatedDelivery = response.EstimatedDelivery
        } else {
            state.ProcessingErrors = append(state.ProcessingErrors, 
                fmt.Sprintf("Shipping failed: %s", response.Message))
        }
        
        // Determine final status
        success := state.IsValidated && 
                  state.InventoryReserved && 
                  state.PaymentProcessed && 
                  state.ShippingArranged
        
        if success {
            // Send success notifications
            completedEvent := &OrderCompletedEvent{
                OrderID:           state.OrderID,
                CustomerID:        state.CustomerID,
                TotalAmount:       state.TotalAmount,
                TrackingNumber:    state.TrackingNumber,
                EstimatedDelivery: state.EstimatedDelivery,
            }
            if err := w.dispatcher.Publish(ctx, completedEvent); err != nil {
                return nil, err
            }
            
            // Send confirmation email
            customerEmail, err := w.getCustomerEmail(ctx, state.CustomerID)
            if err != nil {
                return nil, err
            }
            
            confirmationCmd := &SendOrderConfirmationCommand{
                OrderID:        state.OrderID,
                CustomerID:     state.CustomerID,
                Email:          customerEmail,
                TrackingNumber: state.TrackingNumber,
            }
            if err := w.dispatcher.Send(ctx, "NotificationService", confirmationCmd); err != nil {
                return nil, err
            }
        } else {
            // Handle failure - refund if payment was processed
            if state.PaymentProcessed {
                refundCmd := &RefundPaymentCommand{
                    OrderID:       state.OrderID,
                    TransactionID: state.PaymentTransactionID,
                    Amount:        state.TotalAmount,
                    Reason:        strings.Join(state.ProcessingErrors, "; "),
                }
                if err := w.dispatcher.Send(ctx, "PaymentService", refundCmd); err != nil {
                    return nil, err
                }
            }
            
            // Notify customer of failure
            failedEvent := &OrderFailedEvent{
                OrderID:    state.OrderID,
                CustomerID: state.CustomerID,
                Errors:     state.ProcessingErrors,
            }
            if err := w.dispatcher.Publish(ctx, failedEvent); err != nil {
                return nil, err
            }
        }
        
        status := "Failed"
        if success {
            status = "Completed"
        }
        
        return &OrderFulfillmentResult{
            OrderID:           state.OrderID,
            Success:           success,
            Status:            status,
            TrackingNumber:    state.TrackingNumber,
            EstimatedDelivery: state.EstimatedDelivery,
            Errors:            state.ProcessingErrors,
        }, nil
    })
    
    return workflow
}

func (w *OrderFulfillmentWorkflow) determineWarehouse(address Address) string {
    // Logic to determine warehouse based on shipping address
    return "default-warehouse"
}

func (w *OrderFulfillmentWorkflow) getCustomerTier(ctx context.Context, customerID string) (string, error) {
    // Logic to get customer tier
    return "standard", nil
}

func (w *OrderFulfillmentWorkflow) getCustomerEmail(ctx context.Context, customerID string) (string, error) {
    // Logic to get customer email
    return "customer@example.com", nil
}
```

</td>
</tr>
</table>

## Loan Application Processing

### Scenario: Multi-step Loan Approval Process

Process loan applications through credit check, document verification, underwriting, and approval.

```csharp
// 1. Define loan application state
public class LoanApplicationState
{
    public string ApplicationId { get; set; }
    public string ApplicantId { get; set; }
    public decimal RequestedAmount { get; set; }
    public int TermMonths { get; set; }
    public string Purpose { get; set; }
    
    // Credit check results
    public int CreditScore { get; set; }
    public decimal DebtToIncomeRatio { get; set; }
    public List<string> CreditIssues { get; set; }
    
    // Document verification
    public bool IncomeVerified { get; set; }
    public bool IdentityVerified { get; set; }
    public bool EmploymentVerified { get; set; }
    public List<string> MissingDocuments { get; set; }
    
    // Underwriting results
    public bool UnderwritingApproved { get; set; }
    public decimal ApprovedAmount { get; set; }
    public decimal InterestRate { get; set; }
    public string UnderwriterNotes { get; set; }
    
    // Final decision
    public string Decision { get; set; } // Approved, Denied, Conditional
    public List<string> Conditions { get; set; }
}

// 2. Configure loan processing workflow
public class LoanApplicationWorkflow
{
    private readonly IFlowEndpointFactory _factory;
    
    public void Configure()
    {
        var workflow = _factory.Staged<LoanApplicationRequest, LoanApplicationState>(
            "LoanApplication");
        
        // Stage 1: Initial validation and credit check
        workflow.Stage<LoanApplicationRequest>(async (context, state, request) =>
        {
            state.ApplicationId = Guid.NewGuid().ToString();
            state.ApplicantId = request.ApplicantId;
            state.RequestedAmount = request.RequestedAmount;
            state.TermMonths = request.TermMonths;
            state.Purpose = request.Purpose;
            
            // Initiate credit check
            await context.Request("CreditBureau", new CreditCheckRequest
            {
                ApplicantId = state.ApplicantId,
                ApplicationId = state.ApplicationId,
                ConsentProvided = request.ConsentProvided
            });
            
            // Start document collection process
            await context.Send("DocumentService", new InitiateDocumentCollectionCommand
            {
                ApplicationId = state.ApplicationId,
                ApplicantId = state.ApplicantId,
                RequiredDocuments = DetermineRequiredDocuments(request)
            });
        });
        
        // Stage 2: Process credit report
        workflow.Stage<CreditCheckResponse>(async (context, state, response) =>
        {
            state.CreditScore = response.CreditScore;
            state.DebtToIncomeRatio = response.DebtToIncomeRatio;
            state.CreditIssues = response.Issues;
            
            // Quick denial for very low credit scores
            if (state.CreditScore < 500)
            {
                state.Decision = "Denied";
                state.UnderwriterNotes = "Credit score below minimum threshold";
                return; // Skip to final stage
            }
            
            // Request document verification status
            await context.Request("DocumentService", new GetDocumentStatusRequest
            {
                ApplicationId = state.ApplicationId
            });
        });
        
        // Stage 3: Verify documents
        workflow.Stage<DocumentStatusResponse>(async (context, state, response) =>
        {
            state.IncomeVerified = response.IncomeVerified;
            state.IdentityVerified = response.IdentityVerified;
            state.EmploymentVerified = response.EmploymentVerified;
            state.MissingDocuments = response.MissingDocuments;
            
            // Check if all documents are verified
            if (!response.AllDocumentsVerified)
            {
                // Send reminder for missing documents
                await context.Send("NotificationService", new SendDocumentReminderCommand
                {
                    ApplicantId = state.ApplicantId,
                    ApplicationId = state.ApplicationId,
                    MissingDocuments = state.MissingDocuments
                });
                
                // Set conditional approval
                state.Decision = "Conditional";
                state.Conditions = state.MissingDocuments
                    .Select(d => $"Provide {d}")
                    .ToList();
            }
            
            // Send to underwriting if documents are complete
            if (response.AllDocumentsVerified)
            {
                await context.Request("UnderwritingService", new UnderwritingRequest
                {
                    ApplicationId = state.ApplicationId,
                    CreditScore = state.CreditScore,
                    DebtToIncomeRatio = state.DebtToIncomeRatio,
                    RequestedAmount = state.RequestedAmount,
                    TermMonths = state.TermMonths,
                    VerifiedIncome = response.VerifiedAnnualIncome,
                    EmploymentStatus = response.EmploymentStatus
                });
            }
        });
        
        // Stage 4: Underwriting decision
        workflow.Stage<UnderwritingResponse>(async (context, state, response) =>
        {
            state.UnderwritingApproved = response.Approved;
            state.ApprovedAmount = response.ApprovedAmount;
            state.InterestRate = response.InterestRate;
            state.UnderwriterNotes = response.Notes;
            
            if (response.Approved)
            {
                // Generate loan offer
                await context.Request("LoanOfferService", new GenerateLoanOfferRequest
                {
                    ApplicationId = state.ApplicationId,
                    ApplicantId = state.ApplicantId,
                    ApprovedAmount = state.ApprovedAmount,
                    InterestRate = state.InterestRate,
                    TermMonths = state.TermMonths
                });
            }
            else
            {
                state.Decision = "Denied";
            }
        });
        
        // Final Stage: Complete application and notify
        workflow.LastStage<GenerateLoanOfferResponse, LoanApplicationResult>(
            async (context, state, response) =>
        {
            if (response != null && response.Success)
            {
                state.Decision = "Approved";
                
                // Send approval notification
                await context.Publish(new LoanApprovedEvent
                {
                    ApplicationId = state.ApplicationId,
                    ApplicantId = state.ApplicantId,
                    ApprovedAmount = state.ApprovedAmount,
                    InterestRate = state.InterestRate,
                    OfferExpiryDate = response.OfferExpiryDate
                });
                
                // Send offer documents
                await context.Send("DocumentService", new SendLoanDocumentsCommand
                {
                    ApplicantId = state.ApplicantId,
                    Documents = response.Documents,
                    SigningDeadline = response.OfferExpiryDate
                });
            }
            
            // Record decision
            await context.Send("LoanService", new RecordLoanDecisionCommand
            {
                ApplicationId = state.ApplicationId,
                Decision = state.Decision,
                ApprovedAmount = state.ApprovedAmount,
                InterestRate = state.InterestRate,
                Conditions = state.Conditions,
                UnderwriterNotes = state.UnderwriterNotes
            });
            
            return new LoanApplicationResult
            {
                ApplicationId = state.ApplicationId,
                Decision = state.Decision,
                ApprovedAmount = state.ApprovedAmount,
                InterestRate = state.InterestRate,
                Conditions = state.Conditions,
                NextSteps = GetNextSteps(state)
            };
        });
    }
    
    private List<string> GetNextSteps(LoanApplicationState state)
    {
        return state.Decision switch
        {
            "Approved" => new List<string>
            {
                "Review and sign loan documents",
                "Provide any additional requested information",
                "Schedule closing appointment"
            },
            "Conditional" => state.Conditions,
            "Denied" => new List<string>
            {
                "Review denial reasons",
                "Consider reapplying after addressing issues",
                "Contact loan officer for guidance"
            },
            _ => new List<string>()
        };
    }
}
```

## Document Processing Pipeline

### Scenario: Intelligent Document Processing

Extract, validate, and process documents through OCR, classification, and data extraction.

```csharp
// 1. Document processing state
public class DocumentProcessingState
{
    public string DocumentId { get; set; }
    public string UploadedBy { get; set; }
    public string OriginalFileName { get; set; }
    public long FileSize { get; set; }
    public string MimeType { get; set; }
    
    // Processing stages
    public bool IsScanned { get; set; }
    public string OcrText { get; set; }
    public double OcrConfidence { get; set; }
    
    public string DocumentType { get; set; }
    public double ClassificationConfidence { get; set; }
    
    public Dictionary<string, object> ExtractedData { get; set; }
    public List<ValidationError> ValidationErrors { get; set; }
    
    public bool IsApproved { get; set; }
    public string ReviewedBy { get; set; }
    public string Status { get; set; }
}

// 2. Configure document processing workflow
public class DocumentProcessingWorkflow
{
    private readonly IFlowEndpointFactory _factory;
    
    public void Configure()
    {
        var workflow = _factory.Staged<DocumentUploadRequest, DocumentProcessingState>(
            "DocumentProcessing");
        
        // Stage 1: Pre-process and scan
        workflow.Stage<DocumentUploadRequest>(async (context, state, request) =>
        {
            state.DocumentId = Guid.NewGuid().ToString();
            state.UploadedBy = request.UserId;
            state.OriginalFileName = request.FileName;
            state.FileSize = request.FileSize;
            state.MimeType = request.MimeType;
            
            // Virus scan
            await context.Request("SecurityService", new VirusScanRequest
            {
                DocumentId = state.DocumentId,
                FileUrl = request.FileUrl
            });
        });
        
        // Stage 2: OCR processing
        workflow.Stage<VirusScanResponse>(async (context, state, response) =>
        {
            if (!response.IsClean)
            {
                state.Status = "Rejected";
                state.ValidationErrors = new List<ValidationError>
                {
                    new ValidationError { Field = "File", Message = "Security threat detected" }
                };
                return;
            }
            
            // Perform OCR if needed
            if (IsImageDocument(state.MimeType))
            {
                await context.Request("OcrService", new OcrRequest
                {
                    DocumentId = state.DocumentId,
                    Language = "en",
                    EnhanceImage = true
                });
            }
            else if (state.MimeType == "application/pdf")
            {
                await context.Request("PdfService", new ExtractPdfTextRequest
                {
                    DocumentId = state.DocumentId
                });
            }
        });
        
        // Stage 3: Document classification
        workflow.Stage<OcrResponse>(async (context, state, response) =>
        {
            state.IsScanned = true;
            state.OcrText = response.ExtractedText;
            state.OcrConfidence = response.Confidence;
            
            // Classify document type
            await context.Request("ClassificationService", new ClassifyDocumentRequest
            {
                DocumentId = state.DocumentId,
                Text = state.OcrText,
                FileName = state.OriginalFileName,
                FileSize = state.FileSize
            });
        });
        
        // Stage 4: Extract structured data
        workflow.Stage<ClassifyDocumentResponse>(async (context, state, response) =>
        {
            state.DocumentType = response.DocumentType;
            state.ClassificationConfidence = response.Confidence;
            
            // Route to appropriate extraction service based on type
            switch (state.DocumentType)
            {
                case "Invoice":
                    await context.Request("InvoiceExtractor", new ExtractInvoiceDataRequest
                    {
                        DocumentId = state.DocumentId,
                        OcrText = state.OcrText
                    });
                    break;
                    
                case "Contract":
                    await context.Request("ContractExtractor", new ExtractContractDataRequest
                    {
                        DocumentId = state.DocumentId,
                        OcrText = state.OcrText
                    });
                    break;
                    
                case "Receipt":
                    await context.Request("ReceiptExtractor", new ExtractReceiptDataRequest
                    {
                        DocumentId = state.DocumentId,
                        OcrText = state.OcrText
                    });
                    break;
                    
                default:
                    await context.Request("GenericExtractor", new ExtractGenericDataRequest
                    {
                        DocumentId = state.DocumentId,
                        OcrText = state.OcrText,
                        DocumentType = state.DocumentType
                    });
                    break;
            }
        });
        
        // Stage 5: Validate and enrich data
        workflow.Stage<DataExtractionResponse>(async (context, state, response) =>
        {
            state.ExtractedData = response.ExtractedData;
            
            // Validate extracted data
            var validationRequest = new ValidateExtractedDataRequest
            {
                DocumentId = state.DocumentId,
                DocumentType = state.DocumentType,
                ExtractedData = state.ExtractedData
            };
            
            await context.Request("ValidationService", validationRequest);
        });
        
        // Stage 6: Human review if needed
        workflow.Stage<ValidationResponse>(async (context, state, response) =>
        {
            state.ValidationErrors = response.Errors;
            
            // Determine if human review is needed
            bool needsReview = response.Errors.Any() || 
                              state.OcrConfidence < 0.85 || 
                              state.ClassificationConfidence < 0.90;
            
            if (needsReview)
            {
                await context.Request("ReviewService", new CreateReviewTaskRequest
                {
                    DocumentId = state.DocumentId,
                    DocumentType = state.DocumentType,
                    ValidationErrors = state.ValidationErrors,
                    ExtractedData = state.ExtractedData,
                    Priority = CalculatePriority(state)
                });
            }
            else
            {
                // Auto-approve high confidence documents
                state.IsApproved = true;
                state.ReviewedBy = "System";
            }
        });
        
        // Final Stage: Store and index document
        workflow.LastStage<ReviewTaskResponse, DocumentProcessingResult>(
            async (context, state, response) =>
        {
            if (response != null)
            {
                state.IsApproved = response.Approved;
                state.ReviewedBy = response.ReviewerId;
                state.ExtractedData = response.CorrectedData ?? state.ExtractedData;
            }
            
            state.Status = state.IsApproved ? "Processed" : "Rejected";
            
            if (state.IsApproved)
            {
                // Store in document management system
                await context.Send("DocumentStore", new StoreProcessedDocumentCommand
                {
                    DocumentId = state.DocumentId,
                    DocumentType = state.DocumentType,
                    ExtractedData = state.ExtractedData,
                    Metadata = new DocumentMetadata
                    {
                        UploadedBy = state.UploadedBy,
                        ProcessedAt = DateTime.UtcNow,
                        OcrConfidence = state.OcrConfidence,
                        ReviewedBy = state.ReviewedBy
                    }
                });
                
                // Index for search
                await context.Send("SearchService", new IndexDocumentCommand
                {
                    DocumentId = state.DocumentId,
                    Content = state.OcrText,
                    Metadata = state.ExtractedData
                });
                
                // Trigger downstream processes
                await PublishDocumentEvents(context, state);
            }
            
            return new DocumentProcessingResult
            {
                DocumentId = state.DocumentId,
                Status = state.Status,
                DocumentType = state.DocumentType,
                ExtractedData = state.ExtractedData,
                ValidationErrors = state.ValidationErrors
            };
        });
    }
    
    private async Task PublishDocumentEvents(IStageContext context, DocumentProcessingState state)
    {
        switch (state.DocumentType)
        {
            case "Invoice":
                await context.Publish(new InvoiceProcessedEvent
                {
                    DocumentId = state.DocumentId,
                    VendorName = state.ExtractedData["VendorName"]?.ToString(),
                    InvoiceNumber = state.ExtractedData["InvoiceNumber"]?.ToString(),
                    Amount = Convert.ToDecimal(state.ExtractedData["TotalAmount"]),
                    DueDate = Convert.ToDateTime(state.ExtractedData["DueDate"])
                });
                break;
                
            case "Contract":
                await context.Publish(new ContractProcessedEvent
                {
                    DocumentId = state.DocumentId,
                    ContractType = state.ExtractedData["ContractType"]?.ToString(),
                    Parties = state.ExtractedData["Parties"] as List<string>,
                    EffectiveDate = Convert.ToDateTime(state.ExtractedData["EffectiveDate"]),
                    ExpirationDate = Convert.ToDateTime(state.ExtractedData["ExpirationDate"])
                });
                break;
        }
    }
}
```

## User Onboarding Workflow

### Scenario: Complete User Registration and Verification

Guide new users through registration, verification, and initial setup.

```csharp
// 1. Onboarding state
public class UserOnboardingState
{
    public string UserId { get; set; }
    public string Email { get; set; }
    public string PhoneNumber { get; set; }
    public string FullName { get; set; }
    public DateTime RegistrationDate { get; set; }
    
    // Verification status
    public bool EmailVerified { get; set; }
    public bool PhoneVerified { get; set; }
    public bool IdentityVerified { get; set; }
    
    // KYC/AML
    public string KycStatus { get; set; }
    public List<string> KycDocuments { get; set; }
    public decimal RiskScore { get; set; }
    
    // Account setup
    public bool ProfileCompleted { get; set; }
    public bool PreferencesSet { get; set; }
    public bool InitialFundingComplete { get; set; }
    
    // Referral tracking
    public string ReferralCode { get; set; }
    public string ReferredBy { get; set; }
}

// 2. Configure onboarding workflow
public class UserOnboardingWorkflow
{
    private readonly IFlowEndpointFactory _factory;
    
    public void Configure()
    {
        var workflow = _factory.Staged<UserRegistrationRequest, UserOnboardingState>(
            "UserOnboarding");
        
        // Stage 1: Create account and send verifications
        workflow.Stage<UserRegistrationRequest>(async (context, state, request) =>
        {
            state.UserId = Guid.NewGuid().ToString();
            state.Email = request.Email;
            state.PhoneNumber = request.PhoneNumber;
            state.FullName = request.FullName;
            state.RegistrationDate = DateTime.UtcNow;
            state.ReferralCode = request.ReferralCode;
            
            // Create user account
            await context.Send("UserService", new CreateUserAccountCommand
            {
                UserId = state.UserId,
                Email = state.Email,
                PhoneNumber = state.PhoneNumber,
                FullName = state.FullName
            });
            
            // Send verification emails/SMS
            await context.Send("NotificationService", new SendVerificationEmailCommand
            {
                UserId = state.UserId,
                Email = state.Email,
                VerificationLink = GenerateVerificationLink(state.UserId, "email")
            });
            
            await context.Send("NotificationService", new SendVerificationSmsCommand
            {
                UserId = state.UserId,
                PhoneNumber = state.PhoneNumber,
                VerificationCode = GenerateVerificationCode()
            });
            
            // Check referral if provided
            if (!string.IsNullOrEmpty(state.ReferralCode))
            {
                await context.Request("ReferralService", new ValidateReferralCodeRequest
                {
                    ReferralCode = state.ReferralCode,
                    NewUserId = state.UserId
                });
            }
        });
        
        // Stage 2: Wait for verifications
        workflow.Stage<ValidateReferralCodeResponse>(async (context, state, response) =>
        {
            if (response != null && response.IsValid)
            {
                state.ReferredBy = response.ReferrerUserId;
                
                // Credit referral bonus
                await context.Send("RewardService", new CreditReferralBonusCommand
                {
                    ReferrerUserId = response.ReferrerUserId,
                    ReferredUserId = state.UserId,
                    BonusAmount = response.BonusAmount
                });
            }
            
            // Check verification status
            await context.Request("VerificationService", new GetVerificationStatusRequest
            {
                UserId = state.UserId
            });
        });
        
        // Stage 3: Initiate KYC process
        workflow.Stage<VerificationStatusResponse>(async (context, state, response) =>
        {
            state.EmailVerified = response.EmailVerified;
            state.PhoneVerified = response.PhoneVerified;
            
            // Require at least email verification to proceed
            if (!state.EmailVerified)
            {
                // Send reminder
                await context.Send("NotificationService", new SendVerificationReminderCommand
                {
                    UserId = state.UserId,
                    Email = state.Email,
                    ReminderType = "Email"
                });
                return;
            }
            
            // Start KYC process
            await context.Request("KycService", new InitiateKycRequest
            {
                UserId = state.UserId,
                FullName = state.FullName,
                Email = state.Email,
                PhoneNumber = state.PhoneNumber,
                RequiredLevel = DetermineKycLevel(state)
            });
        });
        
        // Stage 4: Process KYC results
        workflow.Stage<InitiateKycResponse>(async (context, state, response) =>
        {
            state.KycStatus = response.Status;
            state.KycDocuments = response.RequiredDocuments;
            
            if (response.Status == "DocumentsRequired")
            {
                // Request documents
                await context.Send("NotificationService", new RequestKycDocumentsCommand
                {
                    UserId = state.UserId,
                    Email = state.Email,
                    RequiredDocuments = response.RequiredDocuments,
                    UploadLink = response.DocumentUploadUrl
                });
                
                // Wait for document upload
                await context.Request("KycService", new WaitForKycDocumentsRequest
                {
                    UserId = state.UserId,
                    TimeoutMinutes = 10080 // 7 days
                });
            }
            else if (response.Status == "Approved")
            {
                // Proceed to account setup
                await context.Request("AccountService", new GetAccountSetupStatusRequest
                {
                    UserId = state.UserId
                });
            }
        });
        
        // Stage 5: Complete account setup
        workflow.Stage<AccountSetupStatusResponse>(async (context, state, response) =>
        {
            state.ProfileCompleted = response.ProfileCompleted;
            state.PreferencesSet = response.PreferencesSet;
            
            if (!state.ProfileCompleted)
            {
                // Send profile completion reminder
                await context.Send("NotificationService", new SendProfileReminderCommand
                {
                    UserId = state.UserId,
                    Email = state.Email,
                    MissingFields = response.MissingProfileFields
                });
            }
            
            // Check for initial funding (if required)
            await context.Request("PaymentService", new CheckInitialFundingRequest
            {
                UserId = state.UserId,
                MinimumAmount = 10.00m
            });
        });
        
        // Final Stage: Activate account and send welcome package
        workflow.LastStage<CheckInitialFundingResponse, UserOnboardingResult>(
            async (context, state, response) =>
        {
            state.InitialFundingComplete = response?.IsComplete ?? false;
            
            // Determine onboarding status
            var isComplete = state.EmailVerified &&
                           state.KycStatus == "Approved" &&
                           state.ProfileCompleted;
            
            if (isComplete)
            {
                // Activate account
                await context.Send("UserService", new ActivateUserAccountCommand
                {
                    UserId = state.UserId
                });
                
                // Send welcome package
                await context.Publish(new UserOnboardingCompletedEvent
                {
                    UserId = state.UserId,
                    Email = state.Email,
                    FullName = state.FullName,
                    ReferredBy = state.ReferredBy,
                    CompletedAt = DateTime.UtcNow
                });
                
                // Schedule follow-up engagement
                await context.Send("EngagementService", new ScheduleOnboardingFollowUpCommand
                {
                    UserId = state.UserId,
                    FollowUpDays = new[] { 1, 7, 30 }
                });
            }
            
            return new UserOnboardingResult
            {
                UserId = state.UserId,
                Status = isComplete ? "Completed" : "Pending",
                PendingSteps = GetPendingSteps(state),
                NextActions = GetNextActions(state)
            };
        });
    }
}
```

## Expense Approval Workflow

### Scenario: Multi-level Expense Approval

Route expense reports through appropriate approval chains based on amount and policy.

```csharp
// 1. Expense approval state
public class ExpenseApprovalState
{
    public string ExpenseReportId { get; set; }
    public string SubmittedBy { get; set; }
    public string SubmitterDepartment { get; set; }
    public decimal TotalAmount { get; set; }
    public string ExpenseCategory { get; set; }
    public string ProjectCode { get; set; }
    
    // Approval chain
    public List<ApprovalLevel> RequiredApprovals { get; set; }
    public List<ApprovalRecord> Approvals { get; set; } = new List<ApprovalRecord>();
    
    // Policy compliance
    public bool PolicyCompliant { get; set; }
    public List<string> PolicyViolations { get; set; }
    
    // Receipt verification
    public bool ReceiptsVerified { get; set; }
    public List<string> MissingReceipts { get; set; }
    
    // Final status
    public string Status { get; set; }
    public DateTime? ApprovedDate { get; set; }
    public string PaymentStatus { get; set; }
}

// 2. Configure expense approval workflow
public class ExpenseApprovalWorkflow
{
    private readonly IFlowEndpointFactory _factory;
    
    public void Configure()
    {
        var workflow = _factory.Staged<ExpenseReportSubmission, ExpenseApprovalState>(
            "ExpenseApproval");
        
        // Stage 1: Validate and determine approval chain
        workflow.Stage<ExpenseReportSubmission>(async (context, state, request) =>
        {
            state.ExpenseReportId = Guid.NewGuid().ToString();
            state.SubmittedBy = request.EmployeeId;
            state.TotalAmount = request.LineItems.Sum(i => i.Amount);
            state.ExpenseCategory = request.Category;
            state.ProjectCode = request.ProjectCode;
            
            // Get employee details
            var employee = await GetEmployeeDetails(request.EmployeeId);
            state.SubmitterDepartment = employee.Department;
            
            // Check policy compliance
            await context.Request("PolicyService", new CheckExpensePolicyRequest
            {
                ExpenseReportId = state.ExpenseReportId,
                EmployeeId = state.SubmittedBy,
                Department = state.SubmitterDepartment,
                LineItems = request.LineItems,
                TotalAmount = state.TotalAmount
            });
        });
        
        // Stage 2: Verify receipts and determine approvers
        workflow.Stage<CheckExpensePolicyResponse>(async (context, state, response) =>
        {
            state.PolicyCompliant = response.IsCompliant;
            state.PolicyViolations = response.Violations;
            
            // Verify receipts
            await context.Request("ReceiptService", new VerifyReceiptsRequest
            {
                ExpenseReportId = state.ExpenseReportId,
                RequiredReceipts = response.RequiredReceipts
            });
        });
        
        // Stage 3: Route to first approver
        workflow.Stage<VerifyReceiptsResponse>(async (context, state, response) =>
        {
            state.ReceiptsVerified = response.AllReceiptsValid;
            state.MissingReceipts = response.MissingReceipts;
            
            // Determine approval chain based on amount and policy
            state.RequiredApprovals = DetermineApprovalChain(
                state.TotalAmount, 
                state.SubmitterDepartment,
                state.ExpenseCategory);
            
            if (!state.PolicyCompliant || !state.ReceiptsVerified)
            {
                // Reject immediately
                state.Status = "Rejected";
                await NotifyRejection(context, state);
                return;
            }
            
            // Send to first approver
            var firstApprover = state.RequiredApprovals.First();
            await context.Request("ApprovalService", new RequestApprovalRequest
            {
                ExpenseReportId = state.ExpenseReportId,
                ApproverId = firstApprover.ApproverId,
                ApprovalLevel = firstApprover.Level,
                ExpenseDetails = CreateExpenseSummary(state),
                TimeoutDays = 3
            });
        });
        
        // Stage 4: Process approval decisions (recursive)
        workflow.Stage<ApprovalDecisionResponse>(async (context, state, response) =>
        {
            // Record approval
            state.Approvals.Add(new ApprovalRecord
            {
                ApproverId = response.ApproverId,
                Decision = response.Decision,
                Comments = response.Comments,
                ApprovedAt = response.DecisionDate
            });
            
            if (response.Decision == "Rejected")
            {
                state.Status = "Rejected";
                await NotifyRejection(context, state);
                return;
            }
            
            // Check if more approvals needed
            var nextApprovalLevel = GetNextApprovalLevel(state);
            if (nextApprovalLevel != null)
            {
                // Send to next approver
                await context.Request("ApprovalService", new RequestApprovalRequest
                {
                    ExpenseReportId = state.ExpenseReportId,
                    ApproverId = nextApprovalLevel.ApproverId,
                    ApprovalLevel = nextApprovalLevel.Level,
                    ExpenseDetails = CreateExpenseSummary(state),
                    PreviousApprovals = state.Approvals,
                    TimeoutDays = 3
                });
            }
            else
            {
                // All approvals complete
                state.Status = "Approved";
                state.ApprovedDate = DateTime.UtcNow;
                
                // Process payment
                await context.Request("PaymentService", new ProcessExpensePaymentRequest
                {
                    ExpenseReportId = state.ExpenseReportId,
                    EmployeeId = state.SubmittedBy,
                    Amount = state.TotalAmount,
                    PaymentMethod = GetPreferredPaymentMethod(state.SubmittedBy)
                });
            }
        });
        
        // Final Stage: Complete expense processing
        workflow.LastStage<ProcessExpensePaymentResponse, ExpenseApprovalResult>(
            async (context, state, response) =>
        {
            if (response != null)
            {
                state.PaymentStatus = response.Status;
                
                // Notify employee
                await context.Publish(new ExpenseProcessedEvent
                {
                    ExpenseReportId = state.ExpenseReportId,
                    EmployeeId = state.SubmittedBy,
                    Status = state.Status,
                    PaymentStatus = state.PaymentStatus,
                    PaymentDate = response.PaymentDate,
                    PaymentReference = response.PaymentReference
                });
            }
            
            // Update accounting
            if (state.Status == "Approved")
            {
                await context.Send("AccountingService", new RecordExpenseCommand
                {
                    ExpenseReportId = state.ExpenseReportId,
                    Amount = state.TotalAmount,
                    Category = state.ExpenseCategory,
                    ProjectCode = state.ProjectCode,
                    Department = state.SubmitterDepartment,
                    ApprovedBy = state.Approvals.Select(a => a.ApproverId).ToList()
                });
            }
            
            return new ExpenseApprovalResult
            {
                ExpenseReportId = state.ExpenseReportId,
                Status = state.Status,
                ApprovalChain = state.Approvals,
                PaymentStatus = state.PaymentStatus,
                PolicyViolations = state.PolicyViolations
            };
        });
    }
    
    private List<ApprovalLevel> DetermineApprovalChain(
        decimal amount, string department, string category)
    {
        var chain = new List<ApprovalLevel>();
        
        // Direct manager always approves
        chain.Add(new ApprovalLevel 
        { 
            Level = 1, 
            Title = "Direct Manager",
            ApproverId = GetManagerForDepartment(department)
        });
        
        // Department head for amounts > $1000
        if (amount > 1000)
        {
            chain.Add(new ApprovalLevel 
            { 
                Level = 2, 
                Title = "Department Head",
                ApproverId = GetDepartmentHead(department)
            });
        }
        
        // Finance approval for amounts > $5000
        if (amount > 5000)
        {
            chain.Add(new ApprovalLevel 
            { 
                Level = 3, 
                Title = "Finance Director",
                ApproverId = GetFinanceDirector()
            });
        }
        
        // CEO approval for amounts > $25000
        if (amount > 25000)
        {
            chain.Add(new ApprovalLevel 
            { 
                Level = 4, 
                Title = "CEO",
                ApproverId = GetCEO()
            });
        }
        
        return chain;
    }
}
```

## Manufacturing Process Control

### Scenario: Quality Control and Production Tracking

Monitor and control manufacturing processes with quality gates and real-time adjustments.

```csharp
// 1. Manufacturing process state
public class ManufacturingProcessState
{
    public string BatchId { get; set; }
    public string ProductCode { get; set; }
    public int TargetQuantity { get; set; }
    public DateTime StartTime { get; set; }
    
    // Raw materials
    public List<MaterialAllocation> AllocatedMaterials { get; set; }
    public bool MaterialsVerified { get; set; }
    
    // Process stages
    public Dictionary<string, StageResult> StageResults { get; set; } = new();
    
    // Quality control
    public List<QualityCheckResult> QualityChecks { get; set; } = new();
    public bool PassedQualityControl { get; set; }
    
    // Production metrics
    public int ActualQuantity { get; set; }
    public int DefectiveUnits { get; set; }
    public decimal YieldPercentage { get; set; }
    public TimeSpan CycleTime { get; set; }
    
    // Equipment status
    public Dictionary<string, EquipmentStatus> EquipmentStatus { get; set; } = new();
}

// 2. Configure manufacturing workflow
public class ManufacturingProcessWorkflow
{
    private readonly IFlowEndpointFactory _factory;
    
    public void Configure()
    {
        var workflow = _factory.Staged<ProductionOrderRequest, ManufacturingProcessState>(
            "ManufacturingProcess");
        
        // Stage 1: Initialize and allocate materials
        workflow.Stage<ProductionOrderRequest>(async (context, state, request) =>
        {
            state.BatchId = GenerateBatchId();
            state.ProductCode = request.ProductCode;
            state.TargetQuantity = request.Quantity;
            state.StartTime = DateTime.UtcNow;
            
            // Reserve raw materials
            await context.Request("InventoryService", new ReserveMaterialsRequest
            {
                BatchId = state.BatchId,
                ProductCode = state.ProductCode,
                Quantity = state.TargetQuantity,
                RequiredBy = state.StartTime.AddHours(1)
            });
            
            // Prepare equipment
            await context.Send("EquipmentService", new PrepareEquipmentCommand
            {
                BatchId = state.BatchId,
                ProductCode = state.ProductCode,
                RequiredEquipment = GetRequiredEquipment(state.ProductCode)
            });
        });
        
        // Stage 2: Verify materials and start production
        workflow.Stage<ReserveMaterialsResponse>(async (context, state, response) =>
        {
            state.AllocatedMaterials = response.Allocations;
            state.MaterialsVerified = response.Success;
            
            if (!response.Success)
            {
                await context.Publish(new ProductionBlockedEvent
                {
                    BatchId = state.BatchId,
                    Reason = "Insufficient materials",
                    MissingMaterials = response.MissingMaterials
                });
                return;
            }
            
            // Start manufacturing process
            await context.Request("ProductionLine", new StartProductionRequest
            {
                BatchId = state.BatchId,
                ProductCode = state.ProductCode,
                TargetQuantity = state.TargetQuantity,
                Materials = state.AllocatedMaterials,
                ProcessParameters = GetProcessParameters(state.ProductCode)
            });
        });
        
        // Stage 3: Monitor production stages
        workflow.Stage<ProductionStatusUpdate>(async (context, state, update) =>
        {
            // Record stage completion
            state.StageResults[update.StageName] = new StageResult
            {
                Completed = update.Completed,
                Duration = update.Duration,
                UnitsProcessed = update.UnitsProcessed,
                DefectsDetected = update.DefectsDetected
            };
            
            // Real-time quality check
            if (update.RequiresQualityCheck)
            {
                await context.Request("QualityControl", new PerformQualityCheckRequest
                {
                    BatchId = state.BatchId,
                    StageName = update.StageName,
                    SampleSize = CalculateSampleSize(update.UnitsProcessed),
                    CheckType = update.QualityCheckType
                });
            }
            
            // Check if all stages complete
            var productionComplete = IsProductionComplete(state.ProductCode, state.StageResults);
            if (productionComplete)
            {
                // Final quality inspection
                await context.Request("QualityControl", new FinalInspectionRequest
                {
                    BatchId = state.BatchId,
                    ProductCode = state.ProductCode,
                    TotalUnits = state.StageResults.Values.Sum(s => s.UnitsProcessed)
                });
            }
            else
            {
                // Continue to next stage
                var nextStage = GetNextStage(state.ProductCode, state.StageResults);
                await context.Request("ProductionLine", new ContinueProductionRequest
                {
                    BatchId = state.BatchId,
                    NextStage = nextStage
                });
            }
        });
        
        // Stage 4: Process quality results
        workflow.Stage<QualityCheckResult>(async (context, state, result) =>
        {
            state.QualityChecks.Add(result);
            
            // Check if quality standards met
            if (!result.Passed)
            {
                // Determine corrective action
                var action = DetermineCorrectiveAction(result);
                
                switch (action.Type)
                {
                    case "Rework":
                        await context.Send("ProductionLine", new ReworkBatchCommand
                        {
                            BatchId = state.BatchId,
                            Stage = action.TargetStage,
                            Units = result.FailedUnits
                        });
                        break;
                        
                    case "Scrap":
                        state.DefectiveUnits += result.FailedUnits;
                        await context.Send("InventoryService", new ScrapUnitsCommand
                        {
                            BatchId = state.BatchId,
                            Quantity = result.FailedUnits,
                            Reason = result.FailureReason
                        });
                        break;
                        
                    case "AdjustProcess":
                        await context.Send("ProductionLine", new AdjustProcessParametersCommand
                        {
                            BatchId = state.BatchId,
                            Adjustments = action.ParameterAdjustments
                        });
                        break;
                }
            }
        });
        
        // Stage 5: Final inspection and packaging
        workflow.Stage<FinalInspectionResponse>(async (context, state, response) =>
        {
            state.PassedQualityControl = response.Passed;
            state.ActualQuantity = response.PassedUnits;
            state.DefectiveUnits = response.FailedUnits;
            state.YieldPercentage = (decimal)state.ActualQuantity / state.TargetQuantity * 100;
            
            if (state.PassedQualityControl && state.ActualQuantity > 0)
            {
                // Package products
                await context.Request("PackagingService", new PackageProductsRequest
                {
                    BatchId = state.BatchId,
                    ProductCode = state.ProductCode,
                    Quantity = state.ActualQuantity,
                    PackagingSpec = GetPackagingSpec(state.ProductCode)
                });
            }
        });
        
        // Final Stage: Complete production and update systems
        workflow.LastStage<PackagingResponse, ManufacturingResult>(
            async (context, state, response) =>
        {
            state.CycleTime = DateTime.UtcNow - state.StartTime;
            
            // Update inventory
            if (response != null && response.Success)
            {
                await context.Send("InventoryService", new AddFinishedGoodsCommand
                {
                    BatchId = state.BatchId,
                    ProductCode = state.ProductCode,
                    Quantity = state.ActualQuantity,
                    LotNumber = response.LotNumber,
                    ExpiryDate = response.ExpiryDate
                });
            }
            
            // Record production metrics
            await context.Publish(new ProductionCompletedEvent
            {
                BatchId = state.BatchId,
                ProductCode = state.ProductCode,
                TargetQuantity = state.TargetQuantity,
                ActualQuantity = state.ActualQuantity,
                DefectiveUnits = state.DefectiveUnits,
                YieldPercentage = state.YieldPercentage,
                CycleTime = state.CycleTime,
                QualityResults = state.QualityChecks
            });
            
            // Update MES/ERP
            await context.Send("ErpConnector", new UpdateProductionRecordCommand
            {
                BatchId = state.BatchId,
                ProductionData = CreateProductionRecord(state)
            });
            
            // Release equipment
            await context.Send("EquipmentService", new ReleaseEquipmentCommand
            {
                BatchId = state.BatchId,
                Equipment = state.EquipmentStatus.Keys.ToList()
            });
            
            return new ManufacturingResult
            {
                BatchId = state.BatchId,
                Success = state.ActualQuantity > 0,
                ActualQuantity = state.ActualQuantity,
                YieldPercentage = state.YieldPercentage,
                CycleTime = state.CycleTime,
                QualityStatus = state.PassedQualityControl ? "Passed" : "Failed",
                LotNumber = response?.LotNumber
            };
        });
    }
}
```

## Best Practices for StageFlow Workflows

1. **State Management**
   - Keep state objects focused and serializable
   - Store only essential data in state
   - Use external services for large data

2. **Error Handling**
   - Each stage should handle its own errors
   - Use compensation logic for rollbacks
   - Implement circuit breakers for external services

3. **Performance**
   - Use async/await properly
   - Batch operations where possible
   - Set appropriate timeouts for external calls

4. **Monitoring**
   - Publish events at key workflow stages
   - Track timing metrics for each stage
   - Monitor queue depths for stages

5. **Testing**
   - Test each stage independently
   - Test the complete workflow end-to-end
   - Test failure scenarios and compensations

6. **Scalability**
   - Design stages to be idempotent
   - Use queue-based state for resilience
   - Scale stage processors independently