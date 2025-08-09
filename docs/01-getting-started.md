# Getting Started with N8N

Welcome to your automation journey! This guide will help you take your first steps with N8N, from setup to creating your first workflow.

## 📋 Table of Contents
- [What is N8N?](#what-is-n8n)
- [Initial Setup](#initial-setup)
- [Interface Overview](#interface-overview)
- [Your First Workflow](#your-first-workflow)
- [Basic Operations](#basic-operations)
- [Next Steps](#next-steps)

## 🤖 What is N8N?

N8N (pronounced "n-eight-n") is a powerful workflow automation tool that helps you connect different apps and services together. Think of it as a digital assistant that can:

- **Automate repetitive tasks** - No more manual copy-pasting
- **Connect different apps** - Make your tools work together
- **Process data** - Transform and move information between systems
- **Trigger actions** - Respond to events automatically

### Real-World Examples
- 📧 **Email Management**: Automatically save Gmail attachments to Google Drive
- 📱 **Social Media**: Post to multiple platforms simultaneously
- 📊 **Data Sync**: Keep your CRM updated with form submissions
- 🔔 **Notifications**: Get Slack alerts for important events

## 🚀 Initial Setup

### Using Our Docker Setup
We've provided a complete Docker setup for you. Here's how to get started:

1. **Start N8N**:
   ```bash
   cd /path/to/n8n-project
   docker compose up -d
   ```

2. **Access N8N**:
   - Open your browser
   - Go to `http://localhost:5678`
   - Login with username: `admin`, password: `password`

3. **Verify Setup**:
   - You should see the N8N dashboard
   - Check that both N8N and PostgreSQL containers are running

### First-Time Login
When you first access N8N, you'll see:
- 🎨 **Clean, modern interface**
- 🔧 **Setup wizard** (if using cloud)
- 📚 **Tutorial prompts**
- 🎯 **Ready-to-use templates**

## 🎨 Interface Overview

Let's explore the N8N interface together:

### Main Dashboard
```
┌─────────────────────────────────────────────────────┐
│  N8N Logo    [Workflows] [Executions] [Settings]    │
├─────────────────────────────────────────────────────┤
│                                                     │
│  📊 Recent Executions    📈 Workflow Stats          │
│                                                     │
│  🔄 Active Workflows     ⚡ Quick Actions           │
│                                                     │
│  📚 Templates           🎯 Getting Started          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Key Areas
1. **Navigation Bar**: Switch between Workflows, Executions, Settings
2. **Workflow Canvas**: Where you build your automations
3. **Node Panel**: Available integrations and tools
4. **Properties Panel**: Configure selected nodes
5. **Execution Panel**: Test and monitor your workflows

### Essential Buttons
- ➕ **Add Node**: Insert new automation steps
- ▶️ **Execute**: Test your workflow
- 💾 **Save**: Store your changes
- 🔧 **Settings**: Configure workflow properties

## 🎯 Your First Workflow

Let's create a simple workflow to get you started. We'll build a "Welcome Email Sender" that triggers when a webhook is called.

### Step 1: Create New Workflow
1. Click **"New Workflow"** button
2. You'll see an empty canvas with a "Start" node

### Step 2: Add a Webhook Trigger
1. Click the **➕** button next to "Start"
2. Search for **"Webhook"**
3. Click on **"Webhook"** node
4. Configure the webhook:
   ```
   HTTP Method: POST
   Path: welcome
   Response: JSON
   ```

### Step 3: Add Data Processing
1. Click **➕** after the Webhook node
2. Add a **"Set"** node
3. Configure it to extract data:
   ```
   Name: userName
   Value: {{ $json.name }}
   ```

### Step 4: Add Email Action
1. Add an **"Email"** node (or Gmail if you prefer)
2. Configure email settings:
   ```
   To: {{ $json.email }}
   Subject: Welcome {{ $node["Set"].json.userName }}!
   Text: Thank you for joining us!
   ```

### Step 5: Test Your Workflow
1. Click **"Execute Workflow"**
2. Manually trigger with test data
3. Check the execution results

## 🔧 Basic Operations

### Node Management
- **Adding Nodes**: Click ➕ or drag from node panel
- **Connecting Nodes**: Drag from output to input circles
- **Deleting Nodes**: Select and press Delete
- **Copying Nodes**: Ctrl+C, Ctrl+V

### Workflow Operations
- **Save**: Ctrl+S or click Save button
- **Execute**: Click Execute or F5
- **Activate**: Toggle the switch for live workflows
- **Duplicate**: Copy entire workflows

### Data Flow
Understanding how data moves between nodes:

```
[Trigger] → [Process] → [Action]
   ↓           ↓          ↓
  Data    Transform   Execute
```

Each node receives data from the previous node and can modify it before passing it along.

## 💡 Pro Tips for Beginners

### 1. Start Simple
- Begin with 2-3 node workflows
- Add complexity gradually
- Test frequently

### 2. Use Templates
- Browse the template library
- Modify existing templates
- Learn from community examples

### 3. Understand Data
- Use the "JSON" view to see data structure
- Check what each node outputs
- Use the "Table" view for easier reading

### 4. Test Thoroughly
- Use manual execution first
- Test with real data
- Check error scenarios

### 5. Keep Learning
- Read node documentation
- Watch tutorial videos
- Join the community

## 🏃‍♂️ Quick Wins - 5 Simple Workflows

Here are 5 beginner-friendly workflows you can build today:

### 1. **RSS to Slack** (15 minutes)
```
RSS Feed → Set → Slack
```
Get notified in Slack when new articles are published.

### 2. **Form to Email** (10 minutes)
```
Webhook → Set → Email
```
Send email notifications for form submissions.

### 3. **CSV to Database** (20 minutes)
```
Schedule → Read CSV → MySQL
```
Import CSV data to database daily.

### 4. **Image Backup** (25 minutes)
```
Webhook → Download → Google Drive
```
Backup images to cloud storage automatically.

### 5. **Task Reminder** (15 minutes)
```
Schedule → HTTP Request → Slack
```
Daily reminders for important tasks.

## 🚧 Common Beginner Mistakes

### 1. Overcomplicating Workflows
❌ **Wrong**: Building 15-node workflows immediately
✅ **Right**: Start with 3-5 nodes, add complexity later

### 2. Ignoring Error Handling
❌ **Wrong**: Assuming everything always works
✅ **Right**: Plan for failures and edge cases

### 3. Not Testing with Real Data
❌ **Wrong**: Only testing with sample data
✅ **Right**: Use real-world data for testing

### 4. Forgetting to Activate
❌ **Wrong**: Building workflows but not activating them
✅ **Right**: Remember to toggle the activation switch

### 5. Not Understanding Data Types
❌ **Wrong**: Treating all data the same way
✅ **Right**: Learn about strings, numbers, arrays, objects

## 🎓 Learning Exercises

Try these hands-on exercises to reinforce your learning:

### Exercise 1: Data Explorer
**Goal**: Understand data flow
**Task**: Create a workflow that receives JSON data and explores its structure
**Tools**: Webhook → Set → No Op
**Time**: 15 minutes

### Exercise 2: API Consumer
**Goal**: Learn external integrations
**Task**: Fetch weather data and format it nicely
**Tools**: HTTP Request → Set → Email
**Time**: 30 minutes

### Exercise 3: Scheduler
**Goal**: Understand time-based triggers
**Task**: Send yourself a daily motivational quote
**Tools**: Cron → HTTP Request → Slack/Email
**Time**: 20 minutes

## 🔍 Troubleshooting Guide

### Common Issues & Solutions

#### Workflow Not Executing
- ✅ Check if workflow is activated
- ✅ Verify trigger configuration
- ✅ Look for error messages in execution log

#### Nodes Not Connecting
- ✅ Ensure output matches input requirements
- ✅ Check data format compatibility
- ✅ Verify node configuration

#### Data Not Flowing
- ✅ Check expression syntax
- ✅ Verify field names match exactly
- ✅ Use JSON view to inspect data

#### Authentication Errors
- ✅ Check credentials configuration
- ✅ Verify API keys and permissions
- ✅ Test connections individually

## 📚 Essential Resources

### Documentation
- [Official N8N Docs](https://docs.n8n.io/)
- [Node Reference](https://docs.n8n.io/nodes/)
- [Expression Reference](https://docs.n8n.io/code-examples/expressions/)

### Community
- [N8N Community Forum](https://community.n8n.io/)
- [Discord Server](https://discord.gg/n8n)
- [Reddit Community](https://reddit.com/r/n8n)

### Learning Materials
- [YouTube Tutorials](https://www.youtube.com/c/n8nio)
- [Template Library](https://n8n.io/workflows/)
- [Blog Posts](https://blog.n8n.io/)

## ✅ Checklist: Am I Ready for Intermediate?

Before moving to the next level, make sure you can:

- [ ] Navigate the N8N interface confidently
- [ ] Create and save basic workflows
- [ ] Connect different types of nodes
- [ ] Execute workflows manually and automatically
- [ ] Use webhook triggers
- [ ] Handle basic data transformation with Set node
- [ ] Send emails or notifications
- [ ] Troubleshoot simple issues
- [ ] Understand the execution log
- [ ] Activate/deactivate workflows

**Score: 8/10 or higher?** You're ready for [Basic Concepts](./02-basic-concepts.md)!

## 🎯 Next Steps

Congratulations on completing the Getting Started guide! Here's your next steps:

1. **Practice**: Build 2-3 simple workflows on your own
2. **Explore**: Try different nodes and triggers
3. **Learn**: Read the [Basic Concepts](./02-basic-concepts.md) guide
4. **Connect**: Join the N8N community for support

Remember: **Automation mastery comes through practice, not perfection.** Start simple, stay curious, and keep building! 🚀

---

**Ready for more?** Continue with [Basic Concepts →](./02-basic-concepts.md)
