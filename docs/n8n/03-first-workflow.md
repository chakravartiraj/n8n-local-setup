# Your First N8N Workflow Tutorial

Ready to build something real? This hands-on tutorial will guide you through creating a complete, practical workflow from scratch. You'll build an automated customer onboarding system that showcases essential N8N concepts.

## 📋 What We're Building

**Project**: Automated Customer Onboarding System
**Functionality**: 
- Receives new customer data via webhook
- Validates and enriches the data
- Sends personalized welcome email
- Creates customer record in database
- Notifies team in Slack

**Skills You'll Learn**:
- Webhook configuration
- Data validation and transformation
- Email automation
- Database operations
- Team notifications
- Error handling

## 🎯 Prerequisites

Before starting, ensure you have:
- ✅ N8N running locally (from our Docker setup)
- ✅ Basic understanding of JSON data
- ✅ Completed the Getting Started guide
- ✅ A test email account (Gmail works great)

## 🚀 Step-by-Step Tutorial

### Step 1: Create and Setup Workflow

1. **Create New Workflow**
   - Click "New Workflow" in N8N
   - Name it: "Customer Onboarding System"
   - Add description: "Automated customer registration and welcome process"

2. **Configure Workflow Settings**
   - Click the workflow settings (gear icon)
   - Set timezone to your local timezone
   - Enable "Save execution progress"
   - Set error handling to "Continue on fail"

### Step 2: Add Webhook Trigger

The webhook will receive new customer registration data.

1. **Add Webhook Node**
   - Delete the default "Manual Trigger"
   - Add a "Webhook" node
   - Configure:
     ```
     HTTP Method: POST
     Path: customer-onboarding
     Response Mode: Respond Immediately
     Response Code: 200
     Response Data: JSON
     ```

2. **Test Webhook Setup**
   - Save the workflow
   - Note the webhook URL (e.g., `http://localhost:5678/webhook/customer-onboarding`)
   - We'll test this later

### Step 3: Add Data Validation

Let's validate incoming customer data to ensure quality.

1. **Add Function Node**
   - Name: "Validate Customer Data"
   - Add this JavaScript code:
   
   ```javascript
   // Get the incoming data
   const customerData = $input.first().json;
   
   // Validation rules
   const errors = [];
   
   // Required fields
   if (!customerData.name || customerData.name.trim() === '') {
     errors.push('Name is required');
   }
   
   if (!customerData.email || !customerData.email.includes('@')) {
     errors.push('Valid email is required');
   }
   
   if (!customerData.company || customerData.company.trim() === '') {
     errors.push('Company name is required');
   }
   
   // Return validation result
   return {
     json: {
       originalData: customerData,
       isValid: errors.length === 0,
       errors: errors,
       validatedAt: new Date().toISOString()
     }
   };
   ```

### Step 4: Add Conditional Logic

We'll handle valid and invalid data differently.

1. **Add IF Node**
   - Name: "Check Validation Result"
   - Condition: `{{ $json.isValid }}`
   - This creates two paths: TRUE (valid data) and FALSE (invalid data)

### Step 5: Handle Invalid Data (Error Path)

For invalid data, we'll log the error and send notification.

1. **Add Set Node** (connect to FALSE output)
   - Name: "Format Error Response"
   - Add these fields:
     ```
     status: error
     message: Validation failed
     errors: {{ $node["Validate Customer Data"].json.errors }}
     timestamp: {{ $now }}
     ```

2. **Add HTTP Response Node**
   - Name: "Send Error Response"
   - Response Code: 400
   - Response Body: `{{ $json }}`

### Step 6: Process Valid Data (Success Path)

For valid data, let's enrich and process it.

1. **Add Set Node** (connect to TRUE output)
   - Name: "Enrich Customer Data"
   - Add these fields:
     ```
     customerId: {{ $now.toString() }}{{ Math.floor(Math.random() * 1000) }}
     name: {{ $node["Validate Customer Data"].json.originalData.name.trim() }}
     email: {{ $node["Validate Customer Data"].json.originalData.email.toLowerCase() }}
     company: {{ $node["Validate Customer Data"].json.originalData.company.trim() }}
     registrationDate: {{ $now }}
     status: active
     welcomeEmailSent: false
     ```

### Step 7: Send Welcome Email

Let's create a personalized welcome email.

1. **Add Set Node**
   - Name: "Prepare Email Content"
   - Add these fields:
     ```
     to: {{ $json.email }}
     subject: Welcome to our platform, {{ $json.name }}!
     htmlBody: <html><body>
       <h2>Welcome {{ $json.name }}!</h2>
       <p>Thank you for joining us from {{ $json.company }}.</p>
       <p>Your customer ID is: <strong>{{ $json.customerId }}</strong></p>
       <p>We're excited to have you on board!</p>
       <br>
       <p>Best regards,<br>The Team</p>
     </body></html>
     textBody: Welcome {{ $json.name }}! Thank you for joining us from {{ $json.company }}. Your customer ID is: {{ $json.customerId }}. We're excited to have you on board!
     ```

2. **Add Email Node**
   - Name: "Send Welcome Email"
   - Configure with your email settings:
     ```
     From Email: your-email@example.com
     To Email: {{ $node["Prepare Email Content"].json.to }}
     Subject: {{ $node["Prepare Email Content"].json.subject }}
     Email Format: HTML
     Text: {{ $node["Prepare Email Content"].json.htmlBody }}
     ```

### Step 8: Save to Database (Simulation)

Since this is a tutorial, we'll simulate database saving with an HTTP request.

1. **Add HTTP Request Node**
   - Name: "Save to Customer Database"
   - Method: POST
   - URL: `https://jsonplaceholder.typicode.com/posts` (test API)
   - Body Parameters:
     ```json
     {
       "customerId": "{{ $node['Enrich Customer Data'].json.customerId }}",
       "name": "{{ $node['Enrich Customer Data'].json.name }}",
       "email": "{{ $node['Enrich Customer Data'].json.email }}",
       "company": "{{ $node['Enrich Customer Data'].json.company }}",
       "registrationDate": "{{ $node['Enrich Customer Data'].json.registrationDate }}",
       "status": "{{ $node['Enrich Customer Data'].json.status }}"
     }
     ```

### Step 9: Team Notification

Let's notify the team about the new customer.

1. **Add Set Node**
   - Name: "Prepare Team Notification"
   - Add fields:
     ```
     message: 🎉 New customer registered!
     customerName: {{ $node["Enrich Customer Data"].json.name }}
     customerEmail: {{ $node["Enrich Customer Data"].json.email }}
     customerCompany: {{ $node["Enrich Customer Data"].json.company }}
     customerId: {{ $node["Enrich Customer Data"].json.customerId }}
     notificationText: New customer {{ $json.customerName }} from {{ $json.customerCompany }} has registered! Customer ID: {{ $json.customerId }}
     ```

2. **Add HTTP Request Node** (for Slack simulation)
   - Name: "Notify Team"
   - Method: POST
   - URL: `https://httpbin.org/post` (test endpoint)
   - Body: `{{ $node["Prepare Team Notification"].json }}`

### Step 10: Final Response

Send a success response back to the webhook caller.

1. **Add Set Node**
   - Name: "Prepare Success Response"
   - Add fields:
     ```
     status: success
     message: Customer registered successfully
     customerId: {{ $node["Enrich Customer Data"].json.customerId }}
     customerName: {{ $node["Enrich Customer Data"].json.name }}
     timestamp: {{ $now }}
     ```

2. **Add HTTP Response Node**
   - Name: "Send Success Response"
   - Response Code: 200
   - Response Body: `{{ $json }}`

## 🧪 Testing Your Workflow

### Test 1: Valid Customer Data

1. **Manual Test Setup**
   - Click "Execute Workflow"
   - Add test data to the webhook node:
   ```json
   {
     "name": "John Doe",
     "email": "john.doe@example.com",
     "company": "Acme Corp"
   }
   ```

2. **Expected Results**
   - ✅ Validation passes
   - ✅ Data gets enriched with customer ID
   - ✅ Email is prepared (check email content)
   - ✅ Database save simulated
   - ✅ Team notification prepared
   - ✅ Success response returned

### Test 2: Invalid Customer Data

1. **Test with Missing Data**
   ```json
   {
     "name": "",
     "email": "invalid-email",
     "company": ""
   }
   ```

2. **Expected Results**
   - ❌ Validation fails
   - ✅ Error response returned with validation errors
   - ✅ No email sent
   - ✅ No database operations

### Test 3: Real Webhook Test

1. **Use curl or Postman**
   ```bash
   curl -X POST http://localhost:5678/webhook/customer-onboarding \
   -H "Content-Type: application/json" \
   -d '{
     "name": "Jane Smith",
     "email": "jane.smith@techcorp.com",
     "company": "Tech Corp"
   }'
   ```

2. **Check Response**
   - Should receive success response with customer ID
   - Check execution log in N8N

## 🔧 Enhancing Your Workflow

### Enhancement 1: Add More Validation

```javascript
// Add to validation function
if (customerData.phone && !/^\+?[\d\s\-\(\)]+$/.test(customerData.phone)) {
  errors.push('Valid phone number required');
}

if (customerData.website && !customerData.website.includes('.')) {
  errors.push('Valid website URL required');
}
```

### Enhancement 2: Add Customer Segmentation

```javascript
// Add customer tier logic
let customerTier = 'standard';
const emailDomain = customerData.email.split('@')[1];

// Enterprise domains get VIP treatment
const enterpriseDomains = ['microsoft.com', 'google.com', 'apple.com'];
if (enterpriseDomains.includes(emailDomain)) {
  customerTier = 'enterprise';
}

return {
  json: {
    ...existingData,
    customerTier: customerTier
  }
};
```

### Enhancement 3: Add Rate Limiting

```javascript
// Simple rate limiting simulation
const requestsThisHour = Math.floor(Math.random() * 100);
if (requestsThisHour > 50) {
  return {
    json: {
      error: 'Rate limit exceeded. Please try again later.',
      rateLimited: true
    }
  };
}
```

## 🐛 Common Issues & Solutions

### Issue 1: Webhook Not Triggering
**Problem**: Webhook URL returns 404
**Solution**: 
- ✅ Ensure workflow is saved
- ✅ Check webhook path configuration
- ✅ Verify N8N is running on correct port

### Issue 2: Email Not Sending
**Problem**: Email node fails
**Solutions**:
- ✅ Configure SMTP settings correctly
- ✅ Use app-specific password for Gmail
- ✅ Check firewall settings
- ✅ Test with a simple email first

### Issue 3: Data Not Flowing Between Nodes
**Problem**: Expressions return undefined
**Solutions**:
- ✅ Check node names match exactly
- ✅ Use JSON view to inspect data structure
- ✅ Verify expression syntax
- ✅ Test expressions in manual execution mode

### Issue 4: Validation Logic Not Working
**Problem**: Invalid data passes validation
**Solutions**:
- ✅ Check JavaScript syntax in Function node
- ✅ Add console.log for debugging
- ✅ Test validation logic separately
- ✅ Verify IF node condition syntax

## 📊 Workflow Performance Analysis

### Execution Metrics to Monitor
- **Response Time**: Aim for <5 seconds
- **Success Rate**: Target 95%+ success
- **Error Types**: Track validation vs system errors
- **Peak Usage**: Monitor for rate limiting needs

### Optimization Tips
1. **Parallel Processing**: Run independent operations simultaneously
2. **Batch Operations**: Group database operations when possible
3. **Caching**: Store frequently accessed data
4. **Error Recovery**: Implement retry logic for transient failures

## 🎓 Learning Outcomes

After completing this tutorial, you should be able to:

### Technical Skills
- ✅ Configure webhooks for real-time triggers
- ✅ Implement data validation with custom JavaScript
- ✅ Use conditional logic for workflow branching
- ✅ Send automated emails with dynamic content
- ✅ Make HTTP requests for external integrations
- ✅ Handle errors gracefully
- ✅ Test workflows thoroughly

### Best Practices
- ✅ Structure workflows for maintainability
- ✅ Use descriptive node names
- ✅ Implement proper error handling
- ✅ Validate input data
- ✅ Test with realistic data
- ✅ Document complex logic

### Debugging Skills
- ✅ Read execution logs effectively
- ✅ Use manual execution for testing
- ✅ Identify data flow issues
- ✅ Troubleshoot expression problems

## 🚀 Next Project Ideas

Ready for more challenges? Try building these workflows:

### 1. **E-commerce Order Processing** (Intermediate)
- Order webhook → Inventory check → Payment processing → Shipping label → Customer notification

### 2. **Support Ticket System** (Intermediate)
- Email trigger → Parse content → Classify urgency → Assign agent → Send acknowledgment

### 3. **Social Media Manager** (Advanced)
- RSS feed → Content curation → Schedule posts → Cross-platform publishing → Analytics

### 4. **Financial Report Generator** (Advanced)
- Database query → Data analysis → Chart generation → PDF creation → Email distribution

## ✅ Completion Checklist

Mark your progress:

- [ ] Successfully created webhook trigger
- [ ] Implemented data validation with JavaScript
- [ ] Added conditional logic (IF node)
- [ ] Configured email sending
- [ ] Simulated database operations
- [ ] Added team notifications
- [ ] Tested with valid data
- [ ] Tested with invalid data
- [ ] Tested real webhook calls
- [ ] Enhanced workflow with additional features
- [ ] Debugged and resolved issues
- [ ] Documented the workflow

**Score: 10/12 or higher?** Congratulations! You're ready for [Core Nodes Guide](./04-core-nodes.md)!

## 🎯 Next Steps

Amazing work! You've built a complete, production-ready workflow. Here's what to do next:

1. **Deploy**: Consider making this workflow live
2. **Monitor**: Set up alerts for failures
3. **Iterate**: Add features based on usage
4. **Learn**: Explore [Core Nodes Guide](./04-core-nodes.md)
5. **Share**: Show your workflow to the community

**Pro Tip**: The best way to learn N8N is by building real workflows that solve actual problems. Start identifying automation opportunities in your daily work! 🎯

---

**Ready for more nodes?** Continue with [Core Nodes Guide →](./04-core-nodes.md)
