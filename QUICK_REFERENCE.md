# Backend Logging Tutorial - Quick Reference Guide

## 🚀 Quick Start Checklist

### **Essential Logging Principles**
- [ ] **Log at appropriate levels** (DEBUG < INFO < WARN < ERROR < FATAL)
- [ ] **Include contextual information** (timestamp, request ID, user ID)
- [ ] **Use structured logging** (JSON format recommended)
- [ ] **Never log sensitive data** (passwords, API keys, PII)
- [ ] **Configure different settings per environment**
- [ ] **Implement log rotation** to manage disk space
- [ ] **Monitor log volume and performance impact**

---

## 📊 Log Levels Reference

| Level | When to Use | Examples |
|-------|-------------|----------|
| **TRACE** | Finest diagnostic info | Function entry/exit, variable values |
| **DEBUG** | Development debugging | SQL queries, algorithm steps |
| **INFO** | General information | Service startup, configuration loaded |
| **WARN** | Warning conditions | Deprecated API usage, retryable errors |
| **ERROR** | Error conditions | Unhandled exceptions, failed operations |
| **FATAL** | Severe errors | System crashes, critical failures |

---

## 🛠️ Winston Quick Setup

### **Basic Installation**
```bash
npm install winston
npm install winston-daily-rotate-file  # For log rotation
```

### **Essential Winston Configuration**
```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: 'my-service' },
  transports: [
    // Error logs
    new winston.transports.File({ 
      filename: 'logs/error.log', 
      level: 'error',
      maxsize: 10485760, // 10MB
      maxFiles: 5
    }),
    // All logs
    new winston.transports.File({ 
      filename: 'logs/combined.log',
      maxsize: 10485760,
      maxFiles: 10
    })
  ]
});

// Console for development
if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.combine(
      winston.format.colorize(),
      winston.format.simple()
    )
  }));
}

module.exports = logger;
```

### **Common Winston Patterns**
```javascript
// 1. Basic logging
logger.info('Server started', { port: 3000 });
logger.error('Database connection failed', { error: err.message });

// 2. Structured logging with metadata
logger.info('User login', {
  userId: '12345',
  ip: req.ip,
  userAgent: req.get('User-Agent')
});

// 3. Child logger for request correlation
const requestLogger = logger.child({ requestId: 'req-123' });
requestLogger.info('Processing request');

// 4. Performance logging
const timer = logger.startTimer();
// ... some operation
timer.done({ message: 'Operation completed' });
```

---

## 🔧 Common Configurations

### **Environment-Based Config**
```javascript
const config = {
  development: {
    level: 'debug',
    format: winston.format.simple(),
    transports: [new winston.transports.Console()]
  },
  production: {
    level: 'info',
    format: winston.format.json(),
    transports: [
      new winston.transports.File({ filename: 'error.log', level: 'error' }),
      new winston.transports.File({ filename: 'combined.log' })
    ]
  }
};

const env = process.env.NODE_ENV || 'development';
const logger = winston.createLogger(config[env]);
```

### **Express.js Integration**
```javascript
const morgan = require('morgan');
const logger = require('./logger');

// Custom morgan stream
const morganStream = {
  write: (message) => {
    logger.info(message.trim());
  }
};

// Morgan middleware
app.use(morgan('combined', { stream: morganStream }));

// Error handling middleware
app.use((err, req, res, next) => {
  logger.error('Unhandled error', {
    error: err.message,
    stack: err.stack,
    url: req.url,
    method: req.method
  });
  res.status(500).json({ error: 'Internal server error' });
});
```

---

## 📝 What to Log - Quick Guide

### **✅ DO Log**
- Application startup/shutdown
- User authentication events
- API requests/responses (sanitized)
- Database operations
- External service calls
- Business logic events
- Configuration changes
- Performance metrics
- Error conditions and exceptions

### **❌ DON'T Log**
- Passwords or authentication tokens
- Credit card numbers or financial data
- Personal identifiable information (PII)
- API keys or secrets
- Large binary data
- Excessive debug info in production
- High-frequency events without sampling

---

## 🚨 Common Pitfalls and Solutions

### **Problem: Sensitive Data in Logs**
```javascript
// ❌ Wrong
logger.info('User login', { password: req.body.password });

// ✅ Correct
logger.info('User login', { 
  username: req.body.username,
  ip: req.ip,
  timestamp: new Date().toISOString()
});
```

### **Problem: Poor Error Logging**
```javascript
// ❌ Wrong
logger.error('Error occurred');

// ✅ Correct
logger.error('Database connection failed', {
  error: err.message,
  stack: err.stack,
  operation: 'user_fetch',
  userId: req.params.id
});
```

### **Problem: No Request Correlation**
```javascript
// ❌ Wrong - No way to trace related logs
logger.info('Request started');
logger.info('Database query');
logger.info('Request completed');

// ✅ Correct - Use correlation IDs
const requestId = req.headers['x-request-id'] || uuid.v4();
const requestLogger = logger.child({ requestId });

requestLogger.info('Request started');
requestLogger.info('Database query');
requestLogger.info('Request completed');
```

---

## 📈 Performance Optimization Tips

### **1. Use Appropriate Log Levels**
```javascript
// ❌ Wrong - Always evaluates expensive operation
logger.debug(`Complex calculation result: ${complexCalculation()}`);

// ✅ Correct - Only evaluate if debug level is enabled
if (logger.isDebugEnabled()) {
  logger.debug(`Complex calculation result: ${complexCalculation()}`);
}
```

### **2. Asynchronous Logging**
```javascript
const logger = winston.createLogger({
  // ... other config
  transports: [
    new winston.transports.File({
      filename: 'app.log',
      // Enable async writing
      options: { flags: 'a' }
    })
  ]
});
```

### **3. Log Sampling for High Volume**
```javascript
let logCounter = 0;
const sampleRate = 100; // Log every 100th event

app.use((req, res, next) => {
  logCounter++;
  if (logCounter % sampleRate === 0) {
    logger.info('Sampled request', { url: req.url });
  }
  next();
});
```

---

## 🔍 Troubleshooting Guide

### **Common Issues**

| Issue | Symptoms | Solution |
|-------|----------|----------|
| **Logs not appearing** | No log output | Check log level configuration |
| **Performance degradation** | Slow response times | Use async logging, reduce log volume |
| **Disk space issues** | Full disk | Implement log rotation |
| **Memory leaks** | Increasing memory usage | Check for unbounded log buffers |
| **Missing context** | Hard to trace issues | Add correlation IDs and metadata |

### **Debug Commands**
```bash
# Check log file sizes
du -sh logs/*

# Monitor log generation in real-time
tail -f logs/combined.log

# Search for specific errors
grep -i "error" logs/combined.log | tail -20

# Check log rotation
ls -la logs/ | grep ".log"
```

---

## 📚 Module Learning Objectives Summary

### **Module 1: Fundamentals**
- Understand logging purpose and importance
- Master log levels and when to use them
- Implement basic logging best practices

### **Module 2: Universal Patterns**
- Implement structured logging
- Set up log rotation and management
- Configure environment-specific logging

### **Module 3: Simple Scenarios**
- Add logging to monolithic applications
- Implement request/response logging
- Handle errors and exceptions properly

### **Module 4: Advanced Scenarios**
- Implement distributed logging
- Set up centralized log aggregation
- Master correlation and tracing

### **Module 5: Node.js & Winston**
- Master Winston configuration and usage
- Implement custom transports and formats
- Optimize logging performance

---

## 🎯 Skill Assessment Checkpoints

### **Beginner Level ⭐**
- [ ] Can set up basic Winston logger
- [ ] Understands when to use different log levels
- [ ] Knows what data to avoid logging
- [ ] Can implement file and console logging

### **Intermediate Level ⭐⭐**
- [ ] Implements structured logging with metadata
- [ ] Sets up environment-specific configurations
- [ ] Uses correlation IDs for request tracking
- [ ] Integrates logging with Express.js applications

### **Advanced Level ⭐⭐⭐**
- [ ] Designs logging architecture for microservices
- [ ] Implements custom Winston transports
- [ ] Optimizes logging performance
- [ ] Sets up centralized logging with ELK stack

### **Expert Level ⭐⭐⭐⭐**
- [ ] Designs enterprise-scale logging solutions
- [ ] Implements compliance-ready audit logging
- [ ] Creates automated log analysis and alerting
- [ ] Mentors others on logging best practices

---

*Keep this reference guide handy throughout your logging journey! 🚀*
