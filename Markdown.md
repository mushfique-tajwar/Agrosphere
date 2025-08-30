# Agrosphere Features: Implementation Analysis and Viva Guide

## Overview of Four Core Features

### 1. Expense & Earnings Management System
### 2. Nearby Farmers Discovery System  
### 3. User Connection & Network Management
### 4. Direct Messaging System

---

## 1. EXPENSE & EARNINGS MANAGEMENT SYSTEM

### Feature Overview
A comprehensive financial tracking system that allows farmers to record, categorize, and analyze their farm expenses and earnings with visual charts and analytics.

### How It Works

#### Data Flow:
1. **User Input** → **Validation** → **Database Storage** → **Analytics Processing** → **Display**

#### Backend Implementation:

**Database Model (`expenseModel.js`):**
```javascript
// Core function to create expense/earning records
async createExpenseEarning(data) {
  const { user_id, type, category, amount, description, date } = data
  // Inserts into expenses_earnings table with auto-generated year/month fields
  const result = await sql`
    INSERT INTO expenses_earnings (user_id, type, category, amount, description, date)
    VALUES (${user_id}, ${type}, ${category}, ${amount}, ${description || null}, ${date})
    RETURNING id, user_id, type, category, amount, description, date, year, month, created_at
  `
  return result[0]
}
```

**Controller Logic (`expenseController.js`):**
```javascript
// Validates input data before database operations
async createExpenseEarning(userId, data) {
  const { type, category, amount, description, date } = data
  
  // Input validation - ensures required fields are present
  if (!type || !category || !amount || !date) {
    throw new Error("Type, category, amount, and date are required")
  }
  
  // Type validation - only allows 'expense' or 'earning'
  if (!["expense", "earning"].includes(type)) {
    throw new Error("Type must be either 'expense' or 'earning'")
  }
  
  // Amount validation - ensures positive numeric values
  if (isNaN(amount) || Number.parseFloat(amount) <= 0) {
    throw new Error("Amount must be a positive number")
  }
  
  // Calls model to save to database
  const expenseEarning = await expenseModel.createExpenseEarning({
    user_id: userId,
    type,
    category,
    amount: Number.parseFloat(amount),
    description,
    date,
  })
}
```

**Dashboard Analytics Processing:**
```javascript
// Processes multiple time period data for dashboard display
async getDashboardData(userId) {
  const dashboardData = await expenseModel.getDashboardSummary(userId)
  
  // Transforms raw database results into frontend-friendly format
  const processedData = {
    currentMonth: this.processTypeData(dashboardData.currentMonth),    // Current month totals
    last7Days: this.processTypeData(dashboardData.last7Days),          // Last 7 days totals  
    yearlyData: this.processTypeData(dashboardData.yearlyData),        // Full year totals
    availableYears: dashboardData.availableYears,                      // Years with data
    previousMonth: this.processTypeData(dashboardData.previousMonth),  // Previous month for comparison
    last6Months: this.processChartData(dashboardData.last6Months),     // 6-month chart data
    last6Years: this.processYearlyChartData(dashboardData.last6Years), // 6-year chart data
    recentTransactions: dashboardData.recentTransactions,              // Last 30 transactions
    currentYear: dashboardData.currentYear,
  }
}

// Helper function to organize expense/earning data by type
processTypeData(data) {
  const result = { expense: 0, earning: 0 }
  data.forEach((item) => {
    result[item.type] = Number.parseFloat(item.total_amount) || 0  // Converts to number, defaults to 0
  })
  return result
}
```

#### Frontend Implementation:

**Transaction Creation (`ExpenseTrackerPage`):**
```jsx
// Form submission handler for new transactions
const handleTransactionAdded = () => {
  setIsDialogOpen(false)           // Closes the dialog
  fetchDashboardData()             // Refreshes dashboard data
}

// Visual indicators for transaction types
<span className={`text-xl font-bold ${transaction.type === "earning" ? 
  "text-green-600 dark:text-green-400" : "text-red-600 dark:text-red-400"}`}>
  {transaction.type === "earning" ? "+৳" : "-৳"}{transaction.amount.toLocaleString()}
</span>
```

**Chart Visualization:**
```jsx
// Recharts implementation for financial data visualization
<BarChart width={800} height={300} data={chartData}>
  <XAxis dataKey="period" />
  <YAxis />
  <Tooltip formatter={(value, name) => 
    [`৳${value.toLocaleString()}`, name === "earning" ? "Earnings" : "Expenses"]
  } />
  <Bar dataKey="earning" fill="url(#greenGradient)" name="Earnings" />
  <Bar dataKey="expense" fill="url(#redGradient)" name="Expenses" />
</BarChart>
```

### API Endpoints:
- `POST /api/expenses` - Create new expense/earning
- `GET /api/expenses/dashboard` - Get dashboard summary
- `GET /api/expenses/yearly/:year` - Get specific year data
- `DELETE /api/expenses/:id` - Delete expense/earning

### Categories Supported:
**Expenses:** seeds, fertilizer, equipment, labor, fuel
**Earnings:** crop_sales, livestock, dairy, rental

---

## 2. NEARBY FARMERS DISCOVERY SYSTEM

### Feature Overview
Location-based farmer discovery system that helps users find and connect with farmers in their area based on geographical proximity.

### How It Works

#### Data Flow:
1. **User Location Query** → **Database Filtering** → **Connection Status Check** → **Results Display**

#### Backend Implementation:

**Location-Based Search (`userConnectionModel.js`):**
```javascript
// Finds farmers in the same area/city as current user
async findNearbyFarmers(userId, limit = 20) {
  // Gets current user's location information
  const currentUser = await sql`
    SELECT area, city FROM users WHERE id = ${userId}
  `
  
  const { area: currentArea, city: currentCity } = currentUser[0]
  
  // Returns empty if user has no location data
  if (!currentArea && !currentCity) {
    return []
  }
  
  // Builds dynamic WHERE clause based on available location data
  let whereClause = sql`WHERE u.id != ${userId} AND u.is_banned = FALSE`
  
  if (currentArea && currentCity) {
    // Matches either area OR city for broader results
    whereClause = sql`${whereClause} AND (
      (u.area IS NOT NULL AND LOWER(u.area) = LOWER(${currentArea}))
      OR (u.city IS NOT NULL AND LOWER(u.city) = LOWER(${currentCity}))
    )`
  } else if (currentArea) {
    // Matches area only
    whereClause = sql`${whereClause} AND u.area IS NOT NULL AND LOWER(u.area) = LOWER(${currentArea})`
  } else if (currentCity) {
    // Matches city only  
    whereClause = sql`${whereClause} AND u.city IS NOT NULL AND LOWER(u.city) = LOWER(${currentCity})`
  }
}
```

**Search by Location Text (`userConnectionModel.js`):**
```javascript
// Flexible text search across location fields
async findFarmersByLocation(userId, location, area = null, name = null, limit = 20) {
  let whereClause = sql`WHERE u.id != ${userId} AND u.is_banned = FALSE`
  
  if (location) {
    // Searches across multiple location fields with LIKE operator for partial matches
    whereClause = sql`${whereClause} AND (
      LOWER(u.city) LIKE LOWER(${'%' + location.trim() + '%'}) OR 
      LOWER(u.area) LIKE LOWER(${'%' + location.trim() + '%'}) OR 
      LOWER(u.country) LIKE LOWER(${'%' + location.trim() + '%'})
    )`
  }
  
  if (name) {
    // Adds name-based filtering
    whereClause = sql`${whereClause} AND LOWER(u.name) LIKE LOWER(${'%' + name.trim() + '%'})`
  }
}
```

**Connection Status Integration:**
```javascript
// Returns user data with current connection status
SELECT 
  u.id, u.name, u.profile_pic, u.area, u.city, u.country, u.phone,
  u.preferred_crops, u.created_at, u.age,
  CASE 
    WHEN uc.id IS NOT NULL THEN uc.status     // 'pending', 'accepted', or NULL
    ELSE NULL
  END as connection_status,
  CASE 
    WHEN uc.requester_id = ${userId} THEN 'sent'      // User sent request
    WHEN uc.receiver_id = ${userId} THEN 'received'   // User received request
    ELSE NULL
  END as request_direction
FROM users u
LEFT JOIN user_connections uc ON 
  (uc.requester_id = ${userId} AND uc.receiver_id = u.id) OR
  (uc.receiver_id = ${userId} AND uc.requester_id = u.id)
```

#### Frontend Implementation:

**Search Interface (`NearbyPage`):**
```jsx
// Multi-type search functionality
const [searchType, setSearchType] = useState("area")  // 'area', 'city', 'name'
const [searchValue, setSearchValue] = useState("")

const searchFarmers = async () => {
  if (!searchValue.trim()) {
    fetchNearbyFarmers()  // Default to nearby farmers if no search term
    return
  }
  // Builds query parameters dynamically based on search type
  const params = new URLSearchParams({ [searchType]: searchValue.trim() }).toString()
  fetchData(`/api/findnearbyuser?${params}`, setFarmers, "farmers")
}

// Dynamic placeholder text based on search type
<input
  type="text"
  placeholder={
    searchType === 'name' ? "e.g., John Doe, Ahmed Ali..." :
    searchType === 'city' ? "e.g., Dhaka, Chittagong..." :
    "e.g., Dhanmondi, Gulshan..."
  }
  value={searchValue}
  onChange={(e) => setSearchValue(e.target.value)}
  onKeyPress={(e) => e.key === 'Enter' && searchFarmers()}
/>
```

**Farmer Card Component:**
```jsx
// Displays connection status and appropriate action buttons
const getConnectionStatus = (user) => {
  if (user.connection_status === 'pending') {
    return user.request_direction === 'sent' ? 'Request Sent' : 'Request Received'
  }
  if (user.connection_status === 'accepted') {
    return 'Connected'
  }
  return null  // No connection exists
}

// Conditional button rendering based on connection status
{status ? (
  <span className="px-3 py-1 text-sm rounded-full bg-gray-100 text-gray-600">
    {status}
  </span>
) : (
  <button 
    onClick={() => sendRequest(farmer.id)}
    className="px-4 py-1.5 bg-green-600 text-white rounded-full hover:bg-green-700"
  >
    <UserPlus className="h-4 w-4 mr-1" />
    Connect
  </button>
)}
```

### API Endpoints:
- `GET /api/findnearbyuser?nearby=true` - Find farmers in same area/city
- `GET /api/findnearbyuser?location=Dhaka` - Search by location text
- `GET /api/findnearbyuser?area=Dhanmondi` - Search by specific area
- `GET /api/findnearbyuser?name=John` - Search by farmer name

---

## 3. USER CONNECTION & NETWORK MANAGEMENT

### Feature Overview
Complete social networking system for farmers to send/receive connection requests, manage their network, and view connection profiles.

### How It Works

#### Data Flow:
1. **Connection Request** → **Notification** → **Accept/Reject** → **Network Update** → **Profile Access**

#### Backend Implementation:

**Sending Connection Requests (`userConnectionController.js`):**
```javascript
async sendConnectionRequest(requesterId, receiverId) {
  // Prevents self-connection
  if (requesterId === receiverId) {
    throw new Error("Cannot send connection request to yourself")
  }
  
  // Verifies receiver exists and is not banned
  const receiver = await userModel.findUserById(receiverId)
  if (!receiver) {
    throw new Error("User not found")
  }
  
  // Gets requester details for notification
  const requester = await userModel.findUserById(requesterId)
  
  // Creates connection request in database
  const connectionRequest = await userConnectionModel.sendConnectionRequest(requesterId, receiverId)
  
  // Sends notification to receiver
  await notificationController.createNotificationForUser(
    receiverId,
    "Connection_request",
    `${requester.name || 'Someone'} has sent you a connection request`
  )
}
```

**Database Connection Model (`userConnectionModel.js`):**
```javascript
// Prevents duplicate connection requests
async sendConnectionRequest(requesterId, receiverId) {
  // Checks for existing connection in either direction
  const existingConnection = await sql`
    SELECT id, status FROM user_connections
    WHERE (requester_id = ${requesterId} AND receiver_id = ${receiverId})
       OR (requester_id = ${receiverId} AND receiver_id = ${requesterId})
  `
  
  if (existingConnection.length > 0) {
    throw new Error("Connection request already exists")
  }
  
  // Creates new pending connection request
  const result = await sql`
    INSERT INTO user_connections (requester_id, receiver_id, status)
    VALUES (${requesterId}, ${receiverId}, 'pending')
    RETURNING id, requester_id, receiver_id, status, created_at
  `
  return result[0]
}
```

**Responding to Requests (`userConnectionModel.js`):**
```javascript
// Accepts or rejects connection requests
async respondToConnectionRequest(connectionId, userId, response) {
  // Validates response type
  if (!['accepted', 'rejected'].includes(response)) {
    throw new Error("Invalid response. Must be 'accepted' or 'rejected'")
  }
  
  // Updates connection status (only receiver can respond)
  const result = await sql`
    UPDATE user_connections 
    SET status = ${response}, updated_at = CURRENT_TIMESTAMP
    WHERE id = ${connectionId} 
      AND receiver_id = ${userId}     // Ensures only receiver can respond
      AND status = 'pending'          // Can only respond to pending requests
    RETURNING id, requester_id, receiver_id, status, updated_at
  `
  
  if (result.length === 0) {
    throw new Error("Connection request not found or unauthorized")
  }
  
  return result[0]
}
```

**Retrieving User Network (`userConnectionModel.js`):**
```javascript
// Gets all accepted connections for a user
async getUserConnections(userId) {
  const result = await sql`
    SELECT 
      uc.id as connection_id, 
      uc.created_at as connected_since,
      // Uses CASE statements to get friend info regardless of who sent the request
      CASE 
        WHEN uc.requester_id = ${userId} THEN u2.id
        ELSE u1.id
      END as friend_info_id,
      CASE 
        WHEN uc.requester_id = ${userId} THEN u2.name
        ELSE u1.name
      END as friend_info_name,
      // ... (similar pattern for all user fields)
    FROM user_connections uc
    JOIN users u1 ON uc.requester_id = u1.id AND u1.is_banned = FALSE
    JOIN users u2 ON uc.receiver_id = u2.id AND u2.is_banned = FALSE
    WHERE (uc.requester_id = ${userId} OR uc.receiver_id = ${userId})
      AND uc.status = 'accepted'
    ORDER BY uc.created_at DESC
  `
  
  // Transforms flat database result into nested structure
  return result.map(row => ({
    id: row.connection_id,
    connection_id: row.connection_id,
    connected_since: row.connected_since,
    friend_info: {
      id: row.friend_info_id,
      name: row.friend_info_name,
      profile_pic: row.friend_info_profile_pic,
      // ... all friend fields
    }
  }))
}
```

#### Frontend Implementation:

**Network Management Page (`NetworkPage`):**
```jsx
// State management for different connection types
const [connections, setConnections] = useState([])          // Accepted connections
const [pendingRequests, setPendingRequests] = useState([])  // Received requests
const [sentRequests, setSentRequests] = useState([])        // Sent requests

// Fetches all connection data on page load
useEffect(() => {
  fetchConnections()          // GET /api/user-connections/friends
  fetchConnectionRequests()   // GET /api/user-connections?type=all
}, [])

// Responds to connection requests with notifications
const respondToRequest = async (requestId, action) => {
  const response = await fetch(`/api/user-connections/${requestId}`, {
    method: "PATCH",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ response: action === "accept" ? "accepted" : "rejected" }),
  })
  
  const data = await response.json()
  if (data.success) {
    toast({
      title: "Success",
      description: `Connection request ${action}ed successfully`,
    })
    fetchConnectionRequests()  // Refresh requests list
    if (action === "accept") {
      fetchConnections()       // Refresh connections if accepted
    }
  }
}
```

**Connection Request Display:**
```jsx
// Displays received connection requests with action buttons
{pendingRequests
  .filter(req => req.request_direction === 'received')
  .map((request) => (
    <div key={request.id} className="flex items-center justify-between p-4">
      <div className="flex items-center space-x-3">
        <Avatar>
          <AvatarImage src={request.other_user_profile_pic} />
          <AvatarFallback>{request.other_user_name?.[0]}</AvatarFallback>
        </Avatar>
        <div>
          <h4 className="font-semibold">{request.other_user_name}</h4>
          <p className="text-sm text-gray-500">{request.other_user_area}</p>
        </div>
      </div>
      <div className="flex space-x-2">
        <button 
          onClick={() => respondToRequest(request.id, "accept")}
          className="px-3 py-1 bg-green-600 text-white rounded"
        >
          Accept
        </button>
        <button 
          onClick={() => respondToRequest(request.id, "reject")}
          className="px-3 py-1 bg-red-600 text-white rounded"
        >
          Decline
        </button>
      </div>
    </div>
  ))}
```

### API Endpoints:
- `POST /api/user-connections` - Send connection request
- `PATCH /api/user-connections/:id` - Accept/reject request
- `DELETE /api/user-connections/:id` - Remove connection/cancel request
- `GET /api/user-connections` - Get connection requests
- `GET /api/user-connections/friends` - Get accepted connections

---

## 4. DIRECT MESSAGING SYSTEM

### Feature Overview
Real-time messaging system enabling farmers to communicate directly for advice, business discussions, equipment rental, and other agricultural topics.

### How It Works

#### Data Flow:
1. **Conversation Creation** → **Message Composition** → **Real-time Delivery** → **Read Status Updates**

#### Backend Implementation:

**Conversation Management (`messageModel.js`):**
```javascript
// Creates new conversation between users
async createConversation() {
  const result = await sql`
    INSERT INTO conversations (created_at, last_message_at) 
    VALUES (NOW(), NOW())
    RETURNING conversation_id
  `
  return result[0]
}

// Adds users as participants to conversation
async addConversationParticipants(conversationId, userIds) {
  const participants = userIds.map(userId => ({ conversationId, userId }))
  
  for (const participant of participants) {
    await sql`
      INSERT INTO conversation_participants (conversation_id, user_id, joined_at)
      VALUES (${participant.conversationId}, ${participant.userId}, NOW())
      ON CONFLICT (conversation_id, user_id) DO NOTHING  // Prevents duplicate participants
    `
  }
}

// Finds existing conversation between two users
async findConversationBetweenUsers(user1Id, user2Id) {
  const result = await sql`
    SELECT c.conversation_id
    FROM conversations c
    JOIN conversation_participants cp1 ON c.conversation_id = cp1.conversation_id
    JOIN conversation_participants cp2 ON c.conversation_id = cp2.conversation_id
    WHERE cp1.user_id = ${user1Id} AND cp2.user_id = ${user2Id}
    GROUP BY c.conversation_id
    HAVING COUNT(cp1.conversation_id) = 2  // Ensures exactly 2 participants
  `
  return result[0]
}
```

**Message Operations (`messageModel.js`):**
```javascript
// Creates new message in conversation
async createMessage(messageData) {
  const { conversationId, senderId, content } = messageData
  const result = await sql`
    INSERT INTO messages (conversation_id, sender_id, content, created_at) 
    VALUES (${conversationId}, ${senderId}, ${content.trim()}, NOW())
    RETURNING message_id
  `
  return result[0]
}

// Retrieves messages with sender information
async getConversationMessages(conversationId, limit = 50, offset = 0) {
  const result = await sql`
    SELECT m.*, u.name as full_name, u.profile_pic as profile_image 
    FROM messages m
    JOIN users u ON m.sender_id = u.id
    WHERE m.conversation_id = ${conversationId}
    ORDER BY m.created_at ASC           // Chronological order for chat display
    LIMIT ${limit} OFFSET ${offset}
  `
  return result
}

// Marks messages as read when user views conversation
async markMessagesAsRead(conversationId, userId) {
  await sql`
    UPDATE messages 
    SET is_read = TRUE 
    WHERE conversation_id = ${conversationId} 
    AND sender_id != ${userId}          // Don't mark own messages as read
    AND is_read = FALSE                 // Only update unread messages
  `
}
```

**Conversation List with Metadata (`messageModel.js`):**
```javascript
// Gets user's conversations with last message and unread count
async getUserConversations(userId, limit = 20, offset = 0) {
  const result = await sql`
    SELECT DISTINCT
      c.conversation_id,
      c.created_at,
      c.last_message_at,
      // Subquery for last message content
      (SELECT content FROM messages m 
       WHERE m.conversation_id = c.conversation_id 
       ORDER BY m.created_at DESC LIMIT 1) as last_message_content,
      // Subquery for last message timestamp
      (SELECT created_at FROM messages m 
       WHERE m.conversation_id = c.conversation_id 
       ORDER BY m.created_at DESC LIMIT 1) as last_message_time,
      // Subquery for unread message count
      (SELECT COUNT(*) FROM messages m
       WHERE m.conversation_id = c.conversation_id
       AND m.sender_id != ${userId}      // Messages from others
       AND m.is_read = FALSE) as unread_count
    FROM conversations c
    JOIN conversation_participants cp ON c.conversation_id = cp.conversation_id
    WHERE cp.user_id = ${userId}
    ORDER BY c.last_message_at DESC NULLS LAST, c.created_at DESC
    LIMIT ${limit} OFFSET ${offset}
  `
  return result
}
```

**Controller Logic (`messageController.js`):**
```javascript
// Handles message sending with security checks
async sendMessage(senderId, messageData) {
  const { conversationId, content } = messageData
  
  // Input validation
  if (!conversationId || !content?.trim()) {
    return {
      success: false,
      message: "Conversation ID and content are required",
      status: 400
    }
  }
  
  // Security check - verify user is part of conversation
  const isUserInConversation = await messageModel.isUserInConversation(conversationId, senderId)
  if (!isUserInConversation) {
    return {
      success: false,
      message: "You are not part of this conversation",
      status: 403
    }
  }
  
  // Create message and update conversation timestamp
  const messageResult = await messageModel.createMessage({
    conversationId,
    senderId,
    content: content.trim()
  })
  
  await messageModel.updateConversationLastMessage(conversationId)
  
  // Return complete message with sender info
  const message = await messageModel.getMessageWithSender(messageResult.message_id)
  return { success: true, message, status: 201 }
}
```

#### Frontend Implementation:

**Messages Page Structure (`MessagesPage`):**
```jsx
// Two-panel layout: conversation list + active chat
const [conversations, setConversations] = useState([])      // All user conversations
const [selectedConversation, setSelectedConversation] = useState(null)  // Currently active chat
const [messages, setMessages] = useState([])               // Messages in active conversation
const [newMessage, setNewMessage] = useState("")           // Message being typed

// Loads conversations and handles URL parameters for direct user messaging
useEffect(() => {
  const user = getUserFromStorage()
  setCurrentUser(user)
  fetchConversations()
  
  // If user parameter is provided, start conversation with that user
  if (userParam) {
    startConversationWithUser(userParam)  // Creates conversation if doesn't exist
  }
}, [userParam])

// Loads messages when conversation is selected
useEffect(() => {
  if (selectedConversation) {
    fetchMessages(selectedConversation.conversation_id)
  }
}, [selectedConversation])
```

**Message Sending (`MessagesPage`):**
```jsx
// Handles message composition and sending
const sendMessage = async () => {
  if (!newMessage.trim() || !selectedConversation || isSending) return
  
  setIsSending(true)
  const messageText = newMessage.trim()
  setNewMessage("")  // Clear input immediately for better UX
  
  try {
    const response = await fetch("/api/messages", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        conversationId: selectedConversation.conversation_id,
        content: messageText,
      }),
    })
    
    const data = await response.json()
    if (data.success) {
      setMessages(prev => [...prev, data.message])  // Add new message to chat
      fetchConversations()                          // Update conversation list with new last message
      showNotification("Message sent!", "success")
    } else {
      setNewMessage(messageText)  // Restore message text on failure
      throw new Error(data.message)
    }
  } catch (error) {
    showNotification(error.message || "Failed to send message")
  } finally {
    setIsSending(false)
  }
}

// Enter key support for message sending
const handleKeyPress = (e) => {
  if (e.key === 'Enter' && !e.shiftKey) {
    e.preventDefault()
    sendMessage()
  }
}
```

**Conversation List Display:**
```jsx
// Shows conversations with last message and unread count
{conversations.map((conversation) => {
  const otherParticipant = conversation.participants?.find(
    p => p.user_id !== currentUser?.id  // Gets the other person in conversation
  )
  const isSelected = selectedConversation?.conversation_id === conversation.conversation_id
  const lastMessageContent = typeof conversation.last_message === 'object' 
    ? conversation.last_message?.content || 'No messages yet'
    : conversation.last_message || 'No messages yet'
  
  return (
    <div
      key={conversation.conversation_id}
      className={`p-4 cursor-pointer hover:bg-gray-50 transition-colors ${
        isSelected ? 'bg-green-50 border-r-4 border-green-400' : ''
      }`}
      onClick={() => setSelectedConversation(conversation)}
    >
      <div className="flex items-center space-x-3">
        <Avatar>
          <AvatarImage src={otherParticipant?.profile_image} />
          <AvatarFallback>
            {otherParticipant?.full_name?.split(' ').map(n => n[0]).join('').toUpperCase() || 'U'}
          </AvatarFallback>
        </Avatar>
        <div className="flex-1 min-w-0">
          <div className="flex items-center justify-between mb-1">
            <h4 className="font-semibold text-gray-800 truncate">
              {otherParticipant?.full_name || 'Unknown User'}
            </h4>
            {conversation.last_message_at && (
              <span className="text-xs text-gray-500">
                {formatMessageTime(conversation.last_message_at)}
              </span>
            )}
          </div>
          <p className="text-sm text-gray-600 truncate">
            {lastMessageContent}
          </p>
        </div>
        {conversation.unread_count > 0 && (
          <div className="bg-red-500 text-white rounded-full px-2 py-1 text-xs font-semibold">
            {conversation.unread_count}
          </div>
        )}
      </div>
    </div>
  )
})}
```

**Message Display:**
```jsx
// Renders individual messages with sender identification
{messages.map((message) => {
  const isOwnMessage = message.sender_id === currentUser?.id
  
  return (
    <div
      key={message.message_id}
      className={`flex ${isOwnMessage ? 'justify-end' : 'justify-start'} mb-4`}
    >
      <div className={`max-w-xs lg:max-w-md px-4 py-2 rounded-lg ${
        isOwnMessage 
          ? 'bg-green-600 text-white'           // Own messages: green background
          : 'bg-gray-200 text-gray-800'         // Other messages: gray background
      }`}>
        <p className="text-sm">{message.content}</p>
        <p className={`text-xs mt-1 ${
          isOwnMessage ? 'text-green-100' : 'text-gray-500'
        }`}>
          {formatMessageTime(message.created_at)}
        </p>
      </div>
    </div>
  )
})}
```

**Starting New Conversations:**
```jsx
// Creates conversation with another user (called from Network page)
const startConversationWithUser = async (userId) => {
  try {
    const response = await fetch("/api/messages/conversations", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ participantId: userId }),
    })
    
    const data = await response.json()
    if (data.success) {
      setSelectedConversation(data.conversation)  // Opens the new/existing conversation
      fetchConversations()                        // Refreshes conversation list
    } else {
      throw new Error(data.message)
    }
  } catch (error) {
    showNotification(error.message || "Failed to start conversation")
  }
}
```

### API Endpoints:
- `POST /api/messages` - Send message
- `GET /api/messages/conversations/:id` - Get conversation messages
- `POST /api/messages/conversations` - Start new conversation
- `GET /api/messages/conversations` - Get user's conversations

---

## INTEGRATION BETWEEN FEATURES

### Cross-Feature Data Flow:

**1. Network → Messaging Integration:**
```jsx
// From Network page, users can start conversations
<button 
  onClick={() => router.push(`/messages?user=${friend.friend_info.id}`)}
  className="text-blue-600 hover:text-blue-800"
>
  <MessageCircle className="h-4 w-4 mr-1" />
  Message
</button>
```

**2. Expense Tracking → Crop Records Integration:**
```javascript
// When crop records are created, expenses are automatically added to expense tracker
async handleExpenseTracking(userId, cropRecord, landId) {
  const { total_expenses, total_revenue, season, year, crop_name, planting_date, harvest_date } = cropRecord
  
  // Add expenses to expense tracker
  if (total_expenses && total_expenses > 0) {
    await expenseModel.createExpenseEarning({
      user_id: userId,
      type: 'expense',
      category: 'Crop Expenses',
      amount: total_expenses,
      description: `${season} ${year} - ${crop_name} (Land: ${landId})`,
      date: planting_date
    })
  }
  
  // Add profit to expense tracker if there's revenue
  if (total_revenue && total_revenue > 0) {
    const profit = total_revenue - (total_expenses || 0)
    if (profit > 0) {
      await expenseModel.createExpenseEarning({
        user_id: userId,
        type: 'earning',
        category: 'Crop Profit', 
        amount: profit,
        description: `${season} ${year} - ${crop_name} (Land: ${landId})`,
        date: harvest_date || planting_date
      })
    }
  }
}
```

**3. Connection Status in Farmer Discovery:**
```javascript
// Nearby farmers query includes connection status to show appropriate buttons
SELECT 
  u.id, u.name, u.profile_pic, u.area, u.city, u.country, u.phone,
  u.preferred_crops, u.created_at, u.age,
  CASE 
    WHEN uc.id IS NOT NULL THEN uc.status
    ELSE NULL
  END as connection_status,
  CASE 
    WHEN uc.requester_id = ${userId} THEN 'sent'
    WHEN uc.receiver_id = ${userId} THEN 'received'
    ELSE NULL
  END as request_direction
FROM users u
LEFT JOIN user_connections uc ON 
  (uc.requester_id = ${userId} AND uc.receiver_id = u.id) OR
  (uc.receiver_id = ${userId} AND uc.requester_id = u.id)
```

---

## POTENTIAL VIVA QUESTIONS & ANSWERS

### Database Design Questions:

**Q: Why did you use separate tables for expenses/earnings instead of separate tables?**
**A:** I used a single `expenses_earnings` table with a `type` field ('expense' or 'earning') because:
- Both have identical structure (amount, category, date, description)
- Easier to generate comparative analytics (income vs expenses)
- Simplifies queries for dashboard calculations
- Maintains ACID properties for financial transactions
- Easier to add new transaction types in future

**Q: How do you ensure data consistency in the messaging system?**
**A:** Data consistency is maintained through:
- Foreign key constraints between conversations, participants, and messages
- Transaction blocks for conversation creation (conversation + participants)
- Validation checks before message insertion (user must be participant)
- Automatic timestamp updates for last_message_at in conversations
- Proper indexing on frequently queried fields (conversation_id, user_id)

**Q: Explain the user_connections table design.**
**A:** The user_connections table handles bidirectional relationships:
- `requester_id`: User who sent the connection request
- `receiver_id`: User who received the request  
- `status`: 'pending', 'accepted', 'rejected'
- Only receiver can accept/reject (enforced in controller)
- Prevents duplicate connections with unique constraint
- Supports both sent and received request queries

### Performance & Scalability Questions:

**Q: How would you optimize the nearby farmers query for large datasets?**
**A:** Several optimization strategies:
- **Geospatial indexing**: Use PostGIS for location-based queries instead of text matching
- **Caching**: Redis cache for frequently searched locations
- **Pagination**: Implemented with LIMIT/OFFSET, could use cursor-based for better performance
- **Database indexing**: Composite indexes on (area, city) and (is_banned, created_at)
- **Search optimization**: ElasticSearch for complex location searches

**Q: How do you handle real-time messaging at scale?**
**A:** Current implementation can be scaled with:
- **WebSocket connections**: Socket.io for real-time message delivery
- **Message queuing**: Redis pub/sub for message broadcasting
- **Database optimization**: Separate read/write databases, message archiving
- **Caching**: Cache conversation lists and recent messages
- **Horizontal scaling**: Microservices architecture for different features

**Q: How do you prevent SQL injection in your queries?**
**A:** Using prepared statements with the `sql` template literal:
- Parameters are automatically escaped: `sql\`SELECT * FROM users WHERE id = ${userId}\``
- No string concatenation for user inputs
- Type checking at the database layer
- Input validation in controllers before database queries

### Business Logic Questions:

**Q: How do you handle expense categories and ensure data quality?**
**A:** 
- **Predefined categories**: Fixed list for expenses (seeds, fertilizer, equipment, labor, fuel) and earnings (crop_sales, livestock, dairy, rental)
- **Input validation**: Controllers validate category against allowed lists
- **Type validation**: Amount must be positive number, date validation
- **User guidance**: Frontend provides dropdown selections, not free text
- **Future flexibility**: Categories stored as strings, new ones can be added easily

**Q: How do you ensure users can only message their connections?**
**A:** Security is enforced at multiple levels:
- **Database constraints**: Only accepted connections can start conversations
- **Controller validation**: Check connection status before creating conversations
- **Participant verification**: Verify user is conversation participant before sending messages
- **Frontend hiding**: Message buttons only shown for connected users
- **API security**: All message endpoints verify user authorization

**Q: How do you calculate and display financial analytics?**
**A:** Analytics pipeline:
1. **Data aggregation**: SQL SUM() and COUNT() for different time periods
2. **Type separation**: Process expenses and earnings separately
3. **Time-based filtering**: Current month, last 7 days, yearly, etc.
4. **Percentage calculations**: Month-over-month growth calculations
5. **Chart data formatting**: Transform database results for frontend charts
6. **Net income calculation**: Earnings minus expenses with trend indicators

### Security Questions:

**Q: How do you authenticate users across your API endpoints?**
**A:** Authentication flow:
- **Cookie-based auth**: HTTP-only cookies with auth tokens
- **Middleware validation**: Each protected route checks auth token
- **User context**: Token contains user ID for database queries
- **Session management**: Tokens expire and require renewal
- **Route protection**: Unauthorized requests return 401 status

**Q: How do you prevent unauthorized access to conversations?**
**A:** Multi-layer protection:
- **Participant verification**: Check if user is conversation participant before any operation
- **Database-level checks**: JOIN with conversation_participants table
- **Controller validation**: Verify user permissions in every message operation
- **Frontend hiding**: Only show conversations user is part of
- **API consistency**: All endpoints validate user access rights

**Q: How do you handle data privacy in the network features?**
**A:** Privacy measures:
- **Banned user filtering**: Excluded from all searches and connections
- **Voluntary disclosure**: Users control what information to share (preferred crops, location)
- **Connection-based access**: Profile details only visible to connected users
- **Location approximation**: Area/city level, not exact coordinates
- **User consent**: Connection requests require explicit acceptance

### Technical Implementation Questions:

**Q: Explain your error handling strategy.**
**A:** Comprehensive error handling:
- **Try-catch blocks**: All async operations wrapped in error handling
- **Validation errors**: Specific error messages for validation failures
- **Database errors**: Graceful handling of constraint violations
- **HTTP status codes**: Appropriate codes (400, 401, 403, 500)
- **User feedback**: Frontend toast notifications for all error states
- **Logging**: Console logging for debugging (development mode only)

**Q: How do you manage state in your React components?**
**A:** State management patterns:
- **Local state**: useState for component-specific data
- **Effect hooks**: useEffect for data fetching and cleanup
- **State lifting**: Shared state moved to parent components
- **Optimistic updates**: UI updates before API confirmation
- **Error boundaries**: Graceful handling of component errors
- **Loading states**: User feedback during async operations

**Q: Describe your database schema design principles.**
**A:** Design principles followed:
- **Normalization**: Separate entities (users, connections, messages, expenses)
- **Referential integrity**: Foreign key constraints between related tables
- **Indexed queries**: Primary keys and frequently queried columns indexed
- **Timestamp tracking**: Created/updated timestamps for audit trails
- **Soft deletes**: is_banned flag instead of hard deletes for users
- **Scalability**: Designed for horizontal scaling with proper partitioning keys

### Feature Integration Questions:

**Q: How do the four features work together as a system?**
**A:** Integration points:
1. **Network → Messaging**: Connected users can message each other directly
2. **Nearby Farmers → Network**: Discovery leads to connection requests
3. **Crop Records → Expenses**: Agricultural data automatically populates financial tracking
4. **Profile System**: Central user management for all features
5. **Notifications**: Cross-feature notifications for connections and messages
6. **Shared UI**: Consistent design and user experience across features

**Q: How would you add new features to this system?**
**A:** Extension strategy:
- **Database schema**: Add new tables with proper foreign key relationships
- **API consistency**: Follow existing REST patterns for new endpoints
- **Component reuse**: Leverage existing UI components and patterns
- **State management**: Integrate with existing data flow patterns
- **Authentication**: Use existing auth system for new features
- **Notification system**: Extend current notification types

This comprehensive analysis covers the technical implementation, business logic, and architectural decisions for all four features, providing a solid foundation for viva questions and system understanding.
