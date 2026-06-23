# 5.3 Backend Implementation

## 5.3.1 Architecture Overview

The backend system implements a **Layered Architecture** with Model-View-Controller (MVC) principles, designed to support a multi-tenant AI-powered customer support platform. The architecture separates concerns into four distinct layers: API Layer (controllers, routes, middleware), Application Layer (services, validators), Domain Layer (models, business entities), and Infrastructure Layer (repositories, external integrations).

This architectural choice is driven by the need for:
- **Scalability**: Clear separation enables independent scaling of components
- **Maintainability**: Each layer has a single responsibility, reducing coupling
- **Multi-tenancy**: Tenant isolation is enforced consistently across layers
- **Real-time capabilities**: Socket.IO integration for live updates
- **AI integration**: Clean separation between business logic and external AI services

The request lifecycle follows: **Client → Middleware → Controller → Service → Repository → Database → Response**, with Socket.IO events emitted for real-time updates.

Key technologies include Node.js with Express, MongoDB with Mongoose, Socket.IO for real-time communication, JWT for authentication, Google Gemini AI for intelligent responses, and HuggingFace for semantic search embeddings.

## 5.3.2 Key Components

### Authentication & Authorization System

**Why this component is important**: Authentication and authorization form the security foundation of the multi-tenant platform, ensuring that users can only access their company's data and perform actions permitted by their role.

**What problem it solves**: Provides secure, stateless authentication across multiple tenant organizations while enforcing granular role-based access control for different user types (agents, team leaders, managers, owners).

**Code Snippet**:

```javascript
const protect = asyncHandler(async (req, res, next) => {
  let token;
  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    throw ApiError.unauthorized('Access denied. No token provided.');
  }

  const decoded = jwt.verify(token, config.jwt.secret);
  const user = await User.findById(decoded.id).select('-passwordHash');

  if (!user || !user.isActive) {
    throw ApiError.unauthorized('User account not found or deactivated.');
  }

  req.user = user;
  req.userId = user._id;
  req.companyId = user.companyId;
  req.userRole = user.role;

  next();
});
```

**Explanation**: The protect middleware implements JWT-based authentication. It extracts the Bearer token, verifies it using the secret, retrieves the associated user, checks account status, and attaches user context to the request. This middleware is applied to protected routes, ensuring that subsequent handlers have access to req.user, req.companyId, and req.userRole without additional database queries. The stateless approach enables horizontal scaling without session storage.

### AI-Powered Message Processor

**Why this component is important**: This is the core intelligence engine of the system, responsible for generating contextual responses, detecting user intent, determining when to escalate to human agents, and automatically creating support tickets.

**What problem it solves**: Enables automated customer support that understands context, retrieves relevant knowledge, and makes intelligent decisions about escalation, reducing agent workload while maintaining service quality.

**Code Snippet**:

```javascript
async processMessage(companyId, sessionId, userMessage, channel = 'web') {
  const session = await chatSessionRepo.findOne({ companyId, sessionId, status: CHAT_STATUS.ACTIVE });

  const [company, user, userTickets] = await Promise.all([
    companyRepo.model.findById(companyId),
    userRepo.model.findById(session.userId).select('name email phone role telegramChatId'),
    ticketRepo.model.find({ companyId, userId: session.userId })
      .sort({ createdAt: -1 }).limit(5).select('ticketNumber category priority status'),
  ]);

  const relevantKnowledge = await this.findRelevantKnowledge(companyId, userMessage);
  const knowledgeContext = relevantKnowledge.map(k => `[${k.item.type}] ${k.item.title}: ${k.item.content}`);

  const aiResult = await getAIResponse({
    systemPrompt,
    messages: session.messages.slice(-10),
    userMessage,
    knowledgeContext,
  });

  session.messages.push({ role: 'assistant', content: aiResult.answer, timestamp: new Date() });

  const needsEscalation = userAsksForAgent || aiResult.shouldEscalate || aiResult.confidence < 0.5;

  if (needsEscalation && !session.summary.linkedTicketId) {
    const ticket = await ticketRepo.create({
      companyId,
      ticketNumber: await this.generateTicketNumber(companyId),
      userId: session.userId,
      channel,
      category: aiResult.category,
      priority: aiResult.priority,
      status: TICKET_STATUS.PENDING,
      context: { sessionId: session.sessionId, lastUserMessage: userMessage, aiSummary: aiResult.answer.substring(0, 300) },
    });
    session.summary.linkedTicketId = ticket._id;
  }

  await session.save();
  return { session, aiResponse: aiResult, ticket, escalated: !!ticket };
}
```

**Explanation**: This method orchestrates the complete AI response pipeline. It retrieves the chat session and related data in parallel using Promise.all for performance. It finds relevant knowledge items using semantic search, constructs a context-aware system prompt, calls the Gemini AI API, and processes the structured response. The escalation logic checks multiple conditions: explicit user requests, AI confidence scores, and intent detection. When escalation is needed, it automatically creates a ticket with the conversation context. This component demonstrates complex business logic coordination across multiple services and external APIs.

### Real-Time Socket.IO Integration

**Why this component is important**: Enables live updates across the platform, ensuring that agents see new tickets immediately, customers receive AI responses in real-time, and dashboards stay synchronized without polling.

**What problem it solves**: Eliminates the need for polling-based updates, reduces server load, and provides instant feedback to users across multiple client types (web, mobile, admin panels).

**Code Snippet**:

```javascript
io.of('/admin').use(async (socket, next) => {
  try {
    const token = socket.handshake.auth.token;
    const decoded = jwt.verify(token, config.jwt.secret);
    const user = await User.findById(decoded.id).select('-passwordHash');

    if (!user || !user.isActive) {
      return next(new Error('Authentication failed'));
    }

    socket.userId = user._id;
    socket.companyId = user.companyId;
    socket.userRole = user.role;
    next();
  } catch (err) {
    next(new Error('Authentication failed'));
  }
});

io.of('/admin').on('connection', (socket) => {
  socket.join(`company:${socket.companyId}`);

  socket.on('ticket:watch', (ticketId) => {
    socket.join(`ticket:${ticketId}`);
  });

  socket.on('disconnect', () => {
    // Cleanup
  });
});
```

**Explanation**: The Socket.IO implementation uses namespaces to separate concerns: `/admin` for agents, `/webchat` for customers, and `/calls` for voice communication. Each namespace has its own authentication middleware that verifies JWT tokens from the handshake. Rooms are used for targeted broadcasting: company rooms for company-wide events, ticket rooms for ticket-specific updates, and session rooms for chat sessions. This architecture enables efficient event delivery without broadcasting to all connected clients. The authentication on connection ensures that only authorized users can join namespaces and rooms.

### Repository Pattern Implementation

**Why this component is important**: Provides a clean abstraction layer between business logic and database operations, enabling easy testing, potential database migration, and consistent data access patterns.

**What problem it solves**: Decouples application logic from MongoDB/Mongoose specifics, allowing services to work with domain objects rather than database queries, and facilitating mocking in unit tests.

**Code Snippet**:

```javascript
class BaseRepository {
  constructor(model) {
    this.model = model;
  }

  async create(data) {
    return await this.model.create(data);
  }

  async findOne(filter, select = '') {
    let query = this.model.findOne(filter);
    if (select) query = query.select(select);
    return await query.exec();
  }

  async findWithPagination(filter = {}, page = 1, limit = 10, options = {}) {
    const skip = (page - 1) * limit;
    const [data, total] = await Promise.all([
      this.find(filter, { ...options, skip, limit }),
      this.count(filter),
    ]);
    return { data, total, page: Number(page), limit: Number(limit), pages: Math.ceil(total / limit) };
  }
}

class TicketRepository extends BaseRepository {
  async findByCompanyAndStatus(companyId, status, page = 1, limit = 20) {
    return this.findWithPagination(
      { companyId, status },
      page,
      limit,
      { sort: { createdAt: -1 }, populate: ['userId', 'assignedTo'] }
    );
  }
}
```

**Explanation**: The BaseRepository provides common CRUD operations that work with any Mongoose model. The findWithPagination method demonstrates optimization by executing data and count queries in parallel using Promise.all, reducing total query time. Domain-specific repositories like TicketRepository extend the base to add domain-specific query methods. This abstraction enables services to work with a consistent interface regardless of the underlying data store, making the system more maintainable and testable. Repositories are instantiated as singletons and exported from a central index file for consistent usage across the application.

## 5.3.3 Critical Workflows

### 5.3.3.1 User Authentication Flow

**Step-by-step explanation**:

1. **Request received**: Client sends POST `/api/v1/auth/login` with email, password, and companySlug
2. **Validation**: Middleware validates request against Joi schema
3. **Controller**: Delegates to AuthService.login()
4. **Service**: Verifies company exists, finds user within company, checks account status, compares password using bcrypt
5. **Token generation**: JWT token created with user ID, company ID, and role
6. **Response**: Returns user data and token to client

**Controller Snippet**:

```javascript
login = this.catchAsync(async (req, res) => {
  const data = await authService.login(req.body);
  this.sendSuccess(res, data, 'Login successful');
});
```

**Service Snippet**:

```javascript
async login({ email, password, companySlug }) {
  const company = await companyRepo.findOne({ slug: companySlug });
  if (!company) throw ApiError.notFound('Company not found');

  const user = await userRepo.findOne({ companyId: company._id, email });
  if (!user || !user.isActive) throw ApiError.unauthorized('Invalid credentials');

  const isMatch = await user.comparePassword(password);
  if (!isMatch) throw ApiError.unauthorized('Invalid credentials');

  user.lastLogin = new Date();
  await user.save();

  const token = generateToken(user);
  return { user: user.toJSON(), token };
}
```

**Explanation**: The authentication flow demonstrates clean separation of concerns. The controller handles HTTP-specific logic, the service contains business rules (company verification, user lookup, password comparison), and repositories handle data access. Password comparison uses bcrypt's secure comparison function. The token is generated with minimal claims (id, companyId, role) to reduce token size while maintaining necessary context. The lastLogin timestamp is updated for analytics and security monitoring.

### 5.3.3.2 AI Chat Response Flow

**Step-by-step explanation**:

1. **Request received**: Client sends POST `/api/v1/chat/sessions/:sessionId/messages` with message content
2. **Authentication**: JWT verified, tenant isolation enforced, permissions checked
3. **Controller**: Calls messageProcessor.processMessage()
4. **Service**: Retrieves session, fetches related data in parallel, finds relevant knowledge via semantic search, constructs system prompt, calls AI API, processes response, checks escalation conditions, creates ticket if needed
5. **Database**: Updates session with AI response, creates ticket if escalated
6. **Real-time**: Emits Socket.IO events to webchat and admin namespaces
7. **Response**: Returns AI response, intent, confidence, and escalation status

**Controller Snippet**:

```javascript
sendMessage = this.catchAsync(async (req, res) => {
  const result = await messageProcessor.processMessage(req.companyId, req.params.sessionId, req.body.content, 'web');

  const io = getIO();
  io.of('/webchat').to(`session:${req.params.sessionId}`).emit('chat:message', {
    message: { role: 'assistant', content: result.aiResponse.answer, timestamp: new Date() },
  });

  if (result.ticket) {
    io.of('/admin').to(`company:${req.companyId}`).emit('ticket:new', { ticket: result.ticket });
  }

  this.sendSuccess(res, {
    message: { role: 'assistant', content: result.aiResponse.answer },
    intent: result.aiResponse.detectedIntent,
    confidence: result.aiResponse.confidence,
    escalated: result.escalated,
  });
});
```

**Service Snippet**:

```javascript
async findRelevantKnowledge(companyId, userMessage) {
  const embedding = await generateEmbedding(userMessage);
  const items = await KnowledgeItem.find({
    companyId,
    isActive: true,
    embeddingVector: { $exists: true, $ne: [] },
  }).select('title content type embeddingVector');

  const similarities = items.map(item => ({
    item,
    score: cosineSimilarity(embedding, item.embeddingVector),
  })).filter(s => s.score > 0.7);

  return similarities.sort((a, b) => b.score - a.score).slice(0, 3);
}
```

**Explanation**: The chat response flow demonstrates the system's core AI capabilities. The controller handles HTTP concerns and real-time event emission. The service orchestrates complex logic: semantic search using vector embeddings, AI API integration, and escalation decision-making. The findRelevantKnowledge method generates an embedding for the user message, retrieves all active knowledge items with embeddings, calculates cosine similarity scores, filters by threshold (0.7), and returns the top 3 matches. This enables the AI to provide contextually relevant responses based on semantic meaning rather than keyword matching. Parallel data fetching using Promise.all optimizes performance by retrieving company, user, and ticket history concurrently.

### 5.3.3.3 AI Response in Chat and Voice

**AI Response in Chat**:

The system generates intelligent responses for chat messages through a sophisticated AI pipeline. When a user sends a message, the system retrieves relevant knowledge items using semantic search, constructs a context-aware system prompt with conversation history and customer profile, and calls the Google Gemini AI API. The AI returns a structured response including the answer, detected intent, confidence score, escalation flag, category, and priority. This enables the system to provide contextual, accurate responses while automatically determining when human intervention is needed.

**Voice Support**:

For voice interactions, the system supports session recording uploads. Agents can upload audio recordings of voice calls, which are stored and linked to the chat session context. The recording URL is stored in the session's context metadata, enabling future reference and analysis. While the system does not currently implement text-to-speech conversion, the architecture supports integration with TTS services like ElevenLabs for future voice response capabilities.

**Code Snippet - Voice Recording Upload**:

```javascript
uploadSessionRecording = this.catchAsync(async (req, res) => {
  const { sessionId } = req.params;

  if (!req.file) {
    throw ApiError.badRequest('No audio file provided');
  }

  const recordingUrl = `/uploads/calls/${req.file.filename}`;

  const session = await chatSessionRepo.findOne({ sessionId, companyId: req.companyId });
  if (!session) {
    throw ApiError.notFound('Session not found');
  }

  session.context = session.context || {};
  session.context.recordingUrl = recordingUrl;
  await session.save();

  this.sendSuccess(res, { session }, 'Session recording uploaded successfully');
});
```

**Explanation**: This method handles voice call recording uploads. It validates that an audio file was provided, generates a URL for the uploaded file, retrieves the associated chat session, and stores the recording URL in the session's context. This enables the system to maintain a complete record of voice interactions alongside chat history, supporting comprehensive customer interaction analysis and quality assurance.

### 5.3.3.4 Chat and Ticket Analysis

**Quality Assurance Analysis**:

The system implements automated quality assurance analysis for support conversations. The QA service uses AI to analyze full conversation journeys, tracking customer emotional state throughout the interaction, evaluating agent performance, and identifying risk flags. The analysis is designed to be strict and evidence-based, considering the entire conversation journey rather than just the outcome. It specifically handles Egyptian Arabic, slang, typos, and informal writing, making it suitable for regional markets.

**Code Snippet - QA System Prompt**:

```javascript
const SYSTEM_PROMPT = `You are a strict, senior Customer Support QA and Conversation Intelligence analyst.

You analyze full customer support conversations including Egyptian Arabic, slang, typos, and informal writing.
You are critical, evidence-based, and NOT lenient. A happy ending does NOT erase a bad start.

CRITICAL: FULL JOURNEY ANALYSIS — NOT JUST THE ENDING

You MUST read every message in chronological order.
You MUST track the customer's emotional state at EACH stage:
  • initial_tone   → what was the customer's tone at the start?
  • journey_shifts → did it escalate? calm down? when and why?
  • final_tone     → how did the conversation end?

A customer who opened with "ايه يعم الخرا ده" (What the f*** is this) or "شغل زباله" (Garbage work)
is an ANGRY customer — even if they thanked the agent at the end.
You MUST reflect that anger in initial_tone and in risk_flags.

AGENT EVALUATION — BE STRICT

Evaluate ONLY the human agent messages (role: "agent").
Ignore bot/assistant/system messages entirely.

A reply of just "hello" to an incoming angry or frustrated customer is a LOW-VALUE reply.
It shows lack of awareness of the customer's situation.

Flag these explicitly:
  • low_value_replies: vague greetings, one-word replies, non-answers
  • missed_empathy: customer expressed frustration and agent ignored it
  • weak_handling: agent moved on without acknowledging the complaint`;
```

**Explanation**: The QA system prompt demonstrates the sophisticated approach to conversation analysis. It explicitly instructs the AI to analyze the full journey rather than just the outcome, to track emotional state changes throughout the conversation, and to be strict in evaluating agent performance. The prompt includes specific examples of Egyptian Arabic profanity and slang, ensuring the AI can accurately identify negative sentiment in regional language. This enables the system to provide meaningful quality assessments that reflect the actual customer experience, not just whether the issue was eventually resolved.

**Analytics Overview**:

The analytics service provides comprehensive insights into platform performance through key performance indicators, heatmaps, and top performers. It calculates metrics such as average first response time, average resolution time, and ticket distribution by status. It generates heatmaps for chat sessions and ticket creation over time, identifies top categories, channels, and customer intents, and ranks agents by resolution performance.

**Code Snippet - Analytics Aggregation**:

```javascript
const [totalSessions, activeSessions, totalTickets, openTickets, inProgressTickets, resolvedTickets] = await Promise.all([
  ChatSession.countDocuments(sessionFilter),
  ChatSession.countDocuments({ ...sessionFilter, status: 'active' }),
  ticketRepo.count(ticketFilter),
  ticketRepo.count({ ...ticketFilter, status: TICKET_STATUS.PENDING }),
  ticketRepo.count({ ...ticketFilter, status: TICKET_STATUS.OPENED }),
  ticketRepo.count({ ...ticketFilter, status: TICKET_STATUS.CLOSED }),
]);

const avgFirstResponseAgg = await ticketRepo.aggregate([
  { $match: { companyId, firstResponseAt: { $ne: null }, ...(hasDateFilter ? { createdAt: dateFilter } : {}) } },
  { $project: { responseTime: { $subtract: ['$firstResponseAt', '$createdAt'] } } },
  { $group: { _id: null, avgTime: { $avg: '$responseTime' } } },
]);

const topAgents = await ticketRepo.aggregate([
  { $match: { companyId, status: TICKET_STATUS.CLOSED, assignedTo: { $ne: null } } },
  { $group: { _id: '$assignedTo', resolvedCount: { $sum: 1 } } },
  { $sort: { resolvedCount: -1 } },
  { $limit: 10 },
  { $lookup: { from: 'users', localField: '_id', foreignField: '_id', as: 'agent' } },
  { $unwind: '$agent' },
  { $project: { agentId: '$_id', name: '$agent.name', email: '$agent.email', resolvedCount: 1 } },
]);
```

**Explanation**: The analytics service uses MongoDB aggregation pipelines to compute complex metrics efficiently. The first aggregation calculates average first response time by subtracting ticket creation time from first response time. The top agents aggregation joins ticket data with user data to identify the most productive agents based on resolved ticket count. These aggregations provide actionable insights for management to optimize team performance and identify training opportunities.

### 5.3.3.5 Agent Ticket Taking and Response Workflow

**Step-by-step explanation**:

1. **Request received**: Agent sends POST `/api/v1/agent/tickets/:ticketId/claim`
2. **Authentication**: JWT verified, role checked (must be agent), tenant isolation enforced
3. **Controller**: Calls agentTicketService.claimTicket()
4. **Service**: Performs atomic findOneAndUpdate to assign ticket, prevents race conditions, logs event
5. **Database**: Updates ticket status to OPENED, sets assignedTo field
6. **Real-time**: Emits Socket.IO event to admin namespace
7. **External**: Sends Telegram notification if ticket channel is Telegram
8. **Response**: Returns updated ticket with populated user and agent details

**Agent Reply Workflow**:

When an agent replies to a ticket, the system updates the ticket with the response, appends the message to the linked chat session if one exists, and sends real-time notifications to the customer through the appropriate channel (web chat or Telegram). This ensures seamless communication regardless of the channel the customer used to initiate the support request.

**Code Snippet - Agent Reply with Channel Notification**:

```javascript
async claimTicket(companyId, ticketId, agentId) {
  const ticket = await Ticket.findOneAndUpdate(
    { _id: ticketId, companyId, assignedTo: null, status: TICKET_STATUS.PENDING },
    { $set: { assignedTo: agentId, status: TICKET_STATUS.OPENED } },
    { new: true }
  ).populate('userId', 'name email phone').populate('assignedTo', 'name email');

  if (!ticket) {
    const existing = await ticketRepo.findOne({ _id: ticketId, companyId });
    if (!existing) throw ApiError.notFound('Ticket not found');
    if (existing.assignedTo) throw ApiError.conflict('Ticket is already assigned to another agent');
    throw ApiError.conflict('Ticket cannot be claimed');
  }

  if (ticket.channel === CHANNELS.TELEGRAM && ticket.userId) {
    try {
      const company = await companyRepo.model.findById(companyId);
      const botToken = company.channelsConfig?.telegram?.botToken;
      const user = await userRepo.model.findById(ticket.userId._id);

      if (user && user.telegramChatId && botToken) {
        await telegramService.sendMessage(
          botToken,
          user.telegramChatId,
          `تم استلام استفسارك/مشكلتك. سيتواصل معك فريق الدعم قريبا. الموظف (${ticket.assignedTo.name}) دخل المحادثة الآن لمساعدتك.`
        );
      }
    } catch (err) {
      console.error('Failed to send Telegram claim notification:', err.message);
    }
  }

  await logEvent({
    companyId,
    eventType: EVENT_TYPES.TICKET_CLAIMED,
    entityType: 'ticket',
    entityId: ticket._id,
    metadata: { agentId, ticketNumber: ticket.ticketNumber },
  });

  return ticket;
}
```

**Explanation**: The claimTicket method demonstrates multi-channel notification logic. After atomically assigning the ticket, it checks if the ticket originated from Telegram. If so, it retrieves the company's bot token and the customer's Telegram chat ID, then sends a notification in Arabic informing the customer that an agent has taken their ticket. This ensures customers are kept informed regardless of the communication channel. The method also logs the claim event for analytics. The atomic update prevents race conditions where two agents might claim the same ticket simultaneously.

### 5.3.3.6 Admin Overview and Important Features

**Admin Dashboard Overview**:

The admin interface provides comprehensive visibility into platform performance through real-time and historical analytics. Administrators can view key performance indicators including total sessions, active sessions, ticket distribution, average response times, and resolution times. The system provides heatmaps showing chat and ticket volume over time, enabling administrators to identify patterns and peak usage periods.

**Important Features**:

**Multi-Channel Support**: The platform seamlessly integrates multiple communication channels including web chat, Telegram, and voice calls. Each channel has specific adapters that convert incoming messages to the internal format and route responses back through the appropriate channel. This unified architecture ensures consistent customer experience regardless of the communication method.

**Real-Time Monitoring**: Through Socket.IO integration, administrators receive live updates on ticket creation, assignment, status changes, and agent activities. This enables proactive management and immediate response to critical issues without relying on polling or manual refreshes.

**Knowledge Base Management**: Administrators can create and manage knowledge base content that the AI uses to generate accurate responses. The system supports semantic search through vector embeddings, enabling the AI to find relevant knowledge based on meaning rather than keywords. Knowledge items can be categorized, tagged, and marked as active or inactive.

**User and Role Management**: The comprehensive RBAC system allows administrators to create users with different roles (agent, team leader, manager, owner) and assign appropriate permissions. Team leaders can be assigned to supervise specific groups of agents, enabling hierarchical management structures.

**Quality Assurance Dashboard**: The automated QA system provides conversation intelligence analysis, identifying risk flags, evaluating agent performance, and tracking customer sentiment. Administrators can view QA results filtered by agent, sentiment, category, and status, enabling targeted coaching and performance improvement.

**Code Snippet - Analytics KPI Calculation**:

```javascript
async getOverview(companyId, { from, to } = {}) {
  const [
    totalSessions,
    activeSessions,
    totalTickets,
    openTickets,
    inProgressTickets,
    resolvedTickets,
  ] = await Promise.all([
    ChatSession.countDocuments(sessionFilter),
    ChatSession.countDocuments({ ...sessionFilter, status: 'active' }),
    ticketRepo.count(ticketFilter),
    ticketRepo.count({ ...ticketFilter, status: TICKET_STATUS.PENDING }),
    ticketRepo.count({ ...ticketFilter, status: TICKET_STATUS.OPENED }),
    ticketRepo.count({ ...ticketFilter, status: TICKET_STATUS.CLOSED }),
  ]);

  return {
    kpis: {
      totalSessions,
      activeSessions,
      totalTickets,
      openTickets,
      inProgressTickets,
      resolvedTickets,
      avgFirstResponseTime,
      avgResolutionTime,
    },
    heatmap: { chats: chatHeatmap, tickets: ticketHeatmap },
    topCategories,
    topChannels,
    topIntents,
    topAgents,
  };
}
```

**Explanation**: The analytics overview method demonstrates the comprehensive data aggregation that powers the admin dashboard. It executes multiple count queries in parallel using Promise.all for performance. The returned data structure includes KPIs for operational metrics, heatmaps for temporal visualization, and ranked lists for categories, channels, intents, and agents. This single API call provides all the data needed for a comprehensive admin dashboard, reducing client-side complexity and ensuring data consistency.

### 5.3.3.7 Ticket Claiming Flow

**Controller Snippet**:

```javascript
claimTicket = this.catchAsync(async (req, res) => {
  const ticket = await agentTicketService.claimTicket(req.companyId, req.params.ticketId, req.userId);

  const io = getIO();
  io.of('/admin').to(`company:${req.companyId}`).emit('ticket:assigned', {
    ticketId: ticket._id,
    ticketNumber: ticket.ticketNumber,
    agentId: req.userId,
    agentName: req.user.name,
  });

  this.sendSuccess(res, { ticket }, 'Ticket claimed');
});
```

**Service Snippet**:

```javascript
async claimTicket(companyId, ticketId, agentId) {
  const ticket = await Ticket.findOneAndUpdate(
    { _id: ticketId, companyId, assignedTo: null, status: TICKET_STATUS.PENDING },
    { $set: { assignedTo: agentId, status: TICKET_STATUS.OPENED } },
    { new: true }
  ).populate('userId', 'name email phone').populate('assignedTo', 'name email');

  if (!ticket) {
    const existing = await ticketRepo.findOne({ _id: ticketId, companyId });
    if (!existing) throw ApiError.notFound('Ticket not found');
    if (existing.assignedTo) throw ApiError.conflict('Ticket already assigned');
    throw ApiError.conflict('Ticket cannot be claimed');
  }

  await logEvent({
    companyId,
    eventType: EVENT_TYPES.TICKET_CLAIMED,
    entityType: 'ticket',
    entityId: ticket._id,
    metadata: { agentId, ticketNumber: ticket.ticketNumber },
  });

  return ticket;
}
```

**Explanation**: The ticket claiming flow demonstrates atomic operations for preventing race conditions. The findOneAndUpdate operation includes all conditions in the query (unassigned, pending status, correct company) and the update in a single atomic operation, ensuring that two agents cannot claim the same ticket simultaneously. The service provides clear error messages for different failure scenarios (not found, already assigned, wrong status). Event logging provides an audit trail for analytics. The controller emits a Socket.IO event to notify all agents in the company of the assignment, enabling real-time dashboard updates without polling.

## 5.3.4 Architectural Patterns

### Repository Pattern

**Why used**: Abstracts data access logic, providing a clean interface between business logic and database operations. Enables easy testing through mocking and potential database migration without changing application code.

**Implementation**:

```javascript
class BaseRepository {
  constructor(model) {
    this.model = model;
  }

  async create(data) {
    return await this.model.create(data);
  }

  async findWithPagination(filter = {}, page = 1, limit = 10, options = {}) {
    const skip = (page - 1) * limit;
    const [data, total] = await Promise.all([
      this.find(filter, { ...options, skip, limit }),
      this.count(filter),
    ]);
    return { data, total, page: Number(page), limit: Number(limit), pages: Math.ceil(total / limit) };
  }
}

export const ticketRepo = new TicketRepository(Ticket);
```

**Benefit in this project**: Services work with domain objects through a consistent interface, reducing coupling to Mongoose specifics. The findWithPagination method optimizes performance by parallelizing data and count queries. Repositories can be easily mocked in unit tests, improving testability.

### Service Layer Pattern

**Why used**: Encapsulates business logic and use case orchestration, keeping controllers thin and focused on HTTP concerns. Enables code reuse across different interfaces (HTTP, WebSocket, CLI).

**Implementation**:

```javascript
class AuthService {
  async register({ companySlug, name, email, password, phone }) {
    const company = await companyRepo.findOne({ slug: companySlug, isActive: true });
    if (!company) throw ApiError.notFound('Company not found or inactive');

    const existingUser = await userRepo.findOne({ companyId: company._id, email });
    if (existingUser) throw ApiError.conflict('User with this email already exists');

    const user = await userRepo.create({
      companyId: company._id, name, email, passwordHash: password, phone, role: ROLES.CUSTOMER,
    });

    const token = generateToken(user);
    return { user: user.toJSON(), token };
  }
}

export default new AuthService();
```

**Benefit in this project**: Complex business rules (company verification, duplicate checking, token generation) are encapsulated in services. Controllers remain simple, delegating to services. The same service methods can be reused across different entry points (HTTP routes, Socket.IO handlers, CLI commands).

### Middleware Pipeline Pattern

**Why used**: Provides a composable way to handle cross-cutting concerns (authentication, authorization, validation, error handling) in a declarative manner.

**Implementation**:

```javascript
router.post('/tickets/:ticketId/claim',
  protect,
  tenantIsolation,
  allowRoles(ROLES.AGENT),
  validate(agentValidator.ticketIdParam),
  agentController.claimTicket
);
```

**Benefit in this project**: Cross-cutting concerns are applied declaratively through middleware composition. Each middleware handles a specific concern independently. The pipeline is easy to understand and modify. Security (authentication, authorization, tenant isolation) is enforced consistently across all protected routes.

## 5.3.5 Cross-Cutting Concerns

### Authentication & Authorization

**Key middleware**:

```javascript
const requirePermission = (resource, action) => {
  return (req, res, next) => {
    const role = req.userRole;
    const permissions = RBAC_MATRIX[role];
    const resourcePermissions = permissions[resource];

    if (!resourcePermissions || !resourcePermissions.includes(action)) {
      throw ApiError.forbidden(
        `Role '${role}' does not have '${action}' permission on '${resource}'`
      );
    }
    next();
  };
};
```

**Explanation**: The RBAC system uses a declarative permission matrix that maps roles to resources and actions. The requirePermission middleware factory creates authorization middleware that checks this matrix. This provides centralized, auditable authorization that is easy to modify without changing code.

### Validation

**Key middleware**:

```javascript
const validate = (schema) => {
  return (req, res, next) => {
    const errors = [];
    ['body', 'query', 'params'].forEach((key) => {
      if (schema[key]) {
        const { error, value } = schema[key].validate(req[key], {
          abortEarly: false,
          stripUnknown: true,
        });
        if (error) {
          error.details.forEach((detail) => {
            errors.push({ field: detail.path.join('.'), message: detail.message });
          });
        } else {
          req[key] = value;
        }
      }
    });

    if (errors.length > 0) {
      throw ApiError.badRequest('Validation failed', errors);
    }
    next();
  };
};
```

**Explanation**: The validation middleware applies Joi schemas to request data, strips unknown fields to prevent mass assignment attacks, and collects all validation errors for comprehensive feedback. This ensures data integrity at the API boundary before business logic execution.

### Error Handling

**Key middleware**:

```javascript
const errorHandler = (err, req, res, _next) => {
  let statusCode = err.statusCode || 500;
  let message = err.message || 'Internal server error';

  if (err.name === 'ValidationError') {
    statusCode = 400;
    message = 'Validation failed';
  }
  if (err.code === 11000) {
    statusCode = 409;
    const field = Object.keys(err.keyValue)[0];
    message = `Duplicate value for field: ${field}`;
  }
  if (err.name === 'JsonWebTokenError') {
    statusCode = 401;
    message = 'Invalid token';
  }

  res.status(statusCode).json({
    success: false,
    message,
    ...(config.env === 'development' && { stack: err.stack }),
  });
};
```

**Explanation**: Centralized error handling converts various error types (Mongoose validation, duplicate keys, JWT errors) into consistent HTTP responses. In development mode, stack traces are included for debugging. This ensures that clients receive well-formatted error responses regardless of where errors originate.

## 5.3.6 Performance & Optimization

### Async Operations

**Parallel data fetching**:

```javascript
const [company, user, userTickets] = await Promise.all([
  companyRepo.model.findById(companyId),
  userRepo.model.findById(session.userId).select('name email phone role telegramChatId'),
  ticketRepo.model.find({ companyId, userId: session.userId })
    .sort({ createdAt: -1 }).limit(5).select('ticketNumber category priority status'),
]);
```

**Explanation**: Using Promise.all for independent database queries reduces total response time by executing queries concurrently rather than sequentially. This pattern is used throughout the application when multiple independent data fetches are needed.

### Database Indexing

**Key indexes**:

```javascript
ticketSchema.index({ companyId: 1, ticketNumber: 1 }, { unique: true });
ticketSchema.index({ companyId: 1, status: 1, createdAt: -1 });
ticketSchema.index({ companyId: 1, assignedTo: 1, status: 1, createdAt: -1 });
```

**Explanation**: Compound indexes optimize common query patterns. The unique index prevents duplicate ticket numbers within a company. The compound indexes on status and createdAt optimize queries for pending/opened tickets sorted by creation time, which are the most common access patterns in the application.

### Pagination

**Implementation**:

```javascript
async findWithPagination(filter = {}, page = 1, limit = 10, options = {}) {
  const skip = (page - 1) * limit;
  const [data, total] = await Promise.all([
    this.find(filter, { ...options, skip, limit }),
    this.count(filter),
  ]);
  return { data, total, pages: Math.ceil(total / limit) };
}
```

**Explanation**: Pagination is implemented consistently across all list endpoints. The method executes data and count queries in parallel using Promise.all, reducing total query time. It returns both the paginated data and metadata, enabling clients to implement pagination controls efficiently.

## 5.3.7 Conclusion

The backend implementation demonstrates a well-architected system designed for scalability, maintainability, and robustness. The layered architecture with clear separation of concerns enables independent development and testing of components. The Repository Pattern and Service Layer Pattern provide clean abstractions that facilitate future modifications and testing.

The system successfully integrates multiple complex features including AI-powered chat with semantic search, multi-channel communication, ticket management, and real-time communication through Socket.IO. The comprehensive RBAC system ensures secure access control across different user roles and company contexts.

Key strengths include:
- **Modularity**: Clear separation between API, application, domain, and infrastructure layers
- **Scalability**: Async operations, database indexing, and pagination support horizontal scaling
- **Security**: JWT authentication, multi-tenant isolation, RBAC authorization, and input validation
- **Real-time Capabilities**: Socket.IO integration for live updates across multiple namespaces
- **AI Integration**: Sophisticated AI pipeline with knowledge retrieval and context-aware responses

The architecture provides a solid foundation for a multi-tenant AI-powered customer support platform, with design decisions that support current requirements while allowing for future growth and feature expansion.
