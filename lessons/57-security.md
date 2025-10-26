# Урок 57: Security Best Practices

## Введение

Security (безопасность) — это не опциональная фича, а фундаментальное требование для любой production системы. Одна уязвимость может привести к утечке миллионов записей пользователей, финансовым потерям, и разрушению репутации компании.

### Почему это важно?

1. **Data Breaches стоят дорого** — средняя стоимость утечки данных $4.45M (IBM, 2023)
2. **Compliance** — GDPR, HIPAA, PCI DSS требуют безопасности
3. **Reputation** — клиенты не доверяют небезопасным сервисам
4. **Legal liability** — компании несут юридическую ответственность за утечки
5. **Ransomware** — атаки вымогателей стали массовыми

### CIA Triad (основа информационной безопасности)

**Confidentiality (Конфиденциальность)** — данные доступны только авторизованным пользователям
**Integrity (Целостность)** — данные не изменены без авторизации
**Availability (Доступность)** — данные доступны когда нужны

---

## Defense in Depth (эшелонированная защита)

Принцип: множество слоёв защиты, чтобы взлом одного слоя не компрометировал всю систему.

```
┌───────────────────────────────────────────┐
│         User / Attacker                   │
└────────────────┬──────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────┐
│  Layer 1: Network (Firewall, VPC)          │
└────────────────┬────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────┐
│  Layer 2: Authentication (OAuth, MFA)       │
└────────────────┬────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────┐
│  Layer 3: Authorization (RBAC, Policies)    │
└────────────────┬────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────┐
│  Layer 4: Application (Input validation)    │
└────────────────┬────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────┐
│  Layer 5: Data (Encryption at rest)         │
└─────────────────────────────────────────────┘
```

---

## 1. Network Security

### VPC Architecture

```
┌──────────────────────────────────────────────────────┐
│                    Internet                          │
└────────────────────────┬─────────────────────────────┘
                         │
                         ▼
              ┌──────────────────┐
              │  Internet Gateway │
              └─────────┬──────────┘
                        │
         ┌──────────────┴──────────────┐
         │                             │
         ▼                             ▼
┌────────────────┐            ┌────────────────┐
│  Public Subnet │            │  Public Subnet │
│   (AZ-1a)      │            │   (AZ-1b)      │
│  NAT Gateway   │            │  NAT Gateway   │
└────────┬───────┘            └────────┬───────┘
         │                             │
         ▼                             ▼
┌────────────────┐            ┌────────────────┐
│ Private Subnet │            │ Private Subnet │
│   (AZ-1a)      │            │   (AZ-1b)      │
│  App Servers   │            │  App Servers   │
└────────┬───────┘            └────────┬───────┘
         │                             │
         ▼                             ▼
┌────────────────┐            ┌────────────────┐
│ Private Subnet │            │ Private Subnet │
│   (AZ-1a)      │            │   (AZ-1b)      │
│   Databases    │            │   Databases    │
└────────────────┘            └────────────────┘
```

**Terraform Example:**

```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "production-vpc"
  }
}

# Public subnets (для Load Balancer)
resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet-${count.index}"
    Tier = "public"
  }
}

# Private subnets (для приложений)
resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 10}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "private-subnet-${count.index}"
    Tier = "private"
  }
}

# Database subnets (изолированные)
resource "aws_subnet" "database" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 20}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "database-subnet-${count.index}"
    Tier = "database"
  }
}

# NAT Gateway (для outbound traffic из private subnets)
resource "aws_nat_gateway" "main" {
  count         = 2
  subnet_id     = aws_subnet.public[count.index].id
  allocation_id = aws_eip.nat[count.index].id

  tags = {
    Name = "nat-gateway-${count.index}"
  }
}

# Route tables
resource "aws_route_table" "private" {
  count  = 2
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = {
    Name = "private-route-table-${count.index}"
  }
}
```

### Security Groups (Firewall Rules)

```hcl
# Load Balancer security group
resource "aws_security_group" "lb" {
  name_prefix = "lb-sg-"
  vpc_id      = aws_vpc.main.id

  # Allow HTTPS from internet
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Allow HTTP (redirect to HTTPS)
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Application security group
resource "aws_security_group" "app" {
  name_prefix = "app-sg-"
  vpc_id      = aws_vpc.main.id

  # Allow traffic only from load balancer
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.lb.id]
  }

  # Allow SSH only from bastion
  ingress {
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Database security group
resource "aws_security_group" "database" {
  name_prefix = "db-sg-"
  vpc_id      = aws_vpc.main.id

  # Allow PostgreSQL only from app servers
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  # No outbound traffic needed
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### WAF (Web Application Firewall)

```hcl
# AWS WAF для защиты от OWASP Top 10
resource "aws_wafv2_web_acl" "main" {
  name  = "production-waf"
  scope = "REGIONAL"

  default_action {
    allow {}
  }

  # Rate limiting
  rule {
    name     = "RateLimitRule"
    priority = 1

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimitRule"
      sampled_requests_enabled   = true
    }
  }

  # SQL Injection protection
  rule {
    name     = "SQLInjectionRule"
    priority = 2

    action {
      block {}
    }

    statement {
      sqli_match_statement {
        field_to_match {
          body {}
        }

        text_transformation {
          priority = 1
          type     = "URL_DECODE"
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "SQLInjectionRule"
      sampled_requests_enabled   = true
    }
  }

  # XSS protection
  rule {
    name     = "XSSRule"
    priority = 3

    action {
      block {}
    }

    statement {
      xss_match_statement {
        field_to_match {
          body {}
        }

        text_transformation {
          priority = 1
          type     = "URL_DECODE"
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "XSSRule"
      sampled_requests_enabled   = true
    }
  }

  # Geo-blocking (block specific countries)
  rule {
    name     = "GeoBlockRule"
    priority = 4

    action {
      block {}
    }

    statement {
      geo_match_statement {
        country_codes = ["CN", "RU", "KP"]  # Block China, Russia, North Korea
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "GeoBlockRule"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "ProductionWAF"
    sampled_requests_enabled   = true
  }
}
```

---

## 2. Authentication and Authorization

### JWT-based Authentication

```javascript
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');

class AuthService {
  constructor(secretKey, refreshSecretKey) {
    this.secretKey = secretKey;
    this.refreshSecretKey = refreshSecretKey;
  }

  // Register user
  async register(email, password) {
    // Hash password with bcrypt (cost factor 12)
    const hashedPassword = await bcrypt.hash(password, 12);

    await db.query(
      'INSERT INTO users (email, password_hash) VALUES ($1, $2)',
      [email, hashedPassword]
    );

    return { success: true };
  }

  // Login
  async login(email, password) {
    const result = await db.query(
      'SELECT id, password_hash FROM users WHERE email = $1',
      [email]
    );

    if (result.rows.length === 0) {
      throw new Error('Invalid credentials');
    }

    const user = result.rows[0];

    // Compare password
    const isValid = await bcrypt.compare(password, user.password_hash);

    if (!isValid) {
      throw new Error('Invalid credentials');
    }

    // Generate access token (short-lived: 15 minutes)
    const accessToken = jwt.sign(
      { userId: user.id, type: 'access' },
      this.secretKey,
      { expiresIn: '15m' }
    );

    // Generate refresh token (long-lived: 7 days)
    const refreshToken = jwt.sign(
      { userId: user.id, type: 'refresh' },
      this.refreshSecretKey,
      { expiresIn: '7d' }
    );

    // Store refresh token in database
    await db.query(
      'INSERT INTO refresh_tokens (user_id, token, expires_at) VALUES ($1, $2, NOW() + INTERVAL \'7 days\')',
      [user.id, refreshToken]
    );

    return { accessToken, refreshToken };
  }

  // Refresh access token
  async refreshAccessToken(refreshToken) {
    try {
      const decoded = jwt.verify(refreshToken, this.refreshSecretKey);

      if (decoded.type !== 'refresh') {
        throw new Error('Invalid token type');
      }

      // Check if refresh token exists in database
      const result = await db.query(
        'SELECT user_id FROM refresh_tokens WHERE token = $1 AND expires_at > NOW()',
        [refreshToken]
      );

      if (result.rows.length === 0) {
        throw new Error('Invalid refresh token');
      }

      // Generate new access token
      const accessToken = jwt.sign(
        { userId: decoded.userId, type: 'access' },
        this.secretKey,
        { expiresIn: '15m' }
      );

      return { accessToken };
    } catch (error) {
      throw new Error('Invalid refresh token');
    }
  }

  // Logout (revoke refresh token)
  async logout(refreshToken) {
    await db.query('DELETE FROM refresh_tokens WHERE token = $1', [refreshToken]);
  }

  // Verify access token middleware
  verifyToken(req, res, next) {
    const authHeader = req.headers.authorization;

    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      return res.status(401).json({ error: 'No token provided' });
    }

    const token = authHeader.substring(7);

    try {
      const decoded = jwt.verify(token, this.secretKey);

      if (decoded.type !== 'access') {
        return res.status(401).json({ error: 'Invalid token type' });
      }

      req.userId = decoded.userId;
      next();
    } catch (error) {
      return res.status(401).json({ error: 'Invalid token' });
    }
  }
}

// Usage
const authService = new AuthService(
  process.env.JWT_SECRET,
  process.env.JWT_REFRESH_SECRET
);

app.post('/auth/register', async (req, res) => {
  const { email, password } = req.body;

  // Validate input
  if (!email || !password) {
    return res.status(400).json({ error: 'Email and password required' });
  }

  if (password.length < 8) {
    return res.status(400).json({ error: 'Password must be at least 8 characters' });
  }

  try {
    await authService.register(email, password);
    res.json({ success: true });
  } catch (error) {
    res.status(500).json({ error: 'Registration failed' });
  }
});

app.post('/auth/login', async (req, res) => {
  const { email, password } = req.body;

  try {
    const { accessToken, refreshToken } = await authService.login(email, password);

    // Set refresh token as httpOnly cookie
    res.cookie('refreshToken', refreshToken, {
      httpOnly: true,
      secure: true,      // HTTPS only
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000  // 7 days
    });

    res.json({ accessToken });
  } catch (error) {
    res.status(401).json({ error: error.message });
  }
});

// Protected route
app.get('/api/profile', authService.verifyToken.bind(authService), async (req, res) => {
  const user = await db.query('SELECT * FROM users WHERE id = $1', [req.userId]);
  res.json(user.rows[0]);
});
```

### Role-Based Access Control (RBAC)

```javascript
class RBACService {
  constructor() {
    this.roles = {
      admin: ['read', 'write', 'delete', 'manage_users'],
      editor: ['read', 'write'],
      viewer: ['read']
    };
  }

  async getUserRoles(userId) {
    const result = await db.query(
      'SELECT role FROM user_roles WHERE user_id = $1',
      [userId]
    );

    return result.rows.map(r => r.role);
  }

  async hasPermission(userId, permission) {
    const userRoles = await this.getUserRoles(userId);

    for (const role of userRoles) {
      const permissions = this.roles[role] || [];
      if (permissions.includes(permission)) {
        return true;
      }
    }

    return false;
  }

  // Middleware для проверки permissions
  requirePermission(permission) {
    return async (req, res, next) => {
      const hasPermission = await this.hasPermission(req.userId, permission);

      if (!hasPermission) {
        return res.status(403).json({ error: 'Insufficient permissions' });
      }

      next();
    };
  }
}

const rbac = new RBACService();

// Usage
app.delete('/api/users/:id',
  authService.verifyToken.bind(authService),
  rbac.requirePermission('manage_users'),
  async (req, res) => {
    await db.query('DELETE FROM users WHERE id = $1', [req.params.id]);
    res.json({ success: true });
  }
);
```

### OAuth 2.0 / OpenID Connect

```javascript
const passport = require('passport');
const GoogleStrategy = require('passport-google-oauth20').Strategy;

passport.use(new GoogleStrategy({
    clientID: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    callbackURL: 'https://api.example.com/auth/google/callback'
  },
  async (accessToken, refreshToken, profile, done) => {
    // Find or create user
    let user = await db.query(
      'SELECT * FROM users WHERE google_id = $1',
      [profile.id]
    );

    if (user.rows.length === 0) {
      user = await db.query(
        'INSERT INTO users (google_id, email, name) VALUES ($1, $2, $3) RETURNING *',
        [profile.id, profile.emails[0].value, profile.displayName]
      );
    }

    return done(null, user.rows[0]);
  }
));

app.get('/auth/google', passport.authenticate('google', {
  scope: ['profile', 'email']
}));

app.get('/auth/google/callback',
  passport.authenticate('google', { session: false }),
  (req, res) => {
    // Generate JWT
    const token = jwt.sign({ userId: req.user.id }, process.env.JWT_SECRET);
    res.redirect(`https://app.example.com/auth/callback?token=${token}`);
  }
);
```

---

## 3. Input Validation and Sanitization

```javascript
const validator = require('validator');
const { body, validationResult } = require('express-validator');

// SQL Injection protection
app.post('/api/users',
  body('email').isEmail().normalizeEmail(),
  body('name').trim().escape().isLength({ min: 2, max: 100 }),
  body('age').optional().isInt({ min: 0, max: 150 }),
  async (req, res) => {
    const errors = validationResult(req);

    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }

    // Safe: using parameterized queries
    await db.query(
      'INSERT INTO users (email, name, age) VALUES ($1, $2, $3)',
      [req.body.email, req.body.name, req.body.age]
    );

    res.json({ success: true });
  }
);

// XSS protection
const sanitizeHtml = require('sanitize-html');

function sanitizeUserInput(html) {
  return sanitizeHtml(html, {
    allowedTags: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
    allowedAttributes: {
      'a': ['href']
    },
    allowedSchemes: ['http', 'https', 'mailto']
  });
}

app.post('/api/posts',
  body('content').customSanitizer(sanitizeUserInput),
  async (req, res) => {
    const { content } = req.body;

    await db.query(
      'INSERT INTO posts (user_id, content) VALUES ($1, $2)',
      [req.userId, content]
    );

    res.json({ success: true });
  }
);

// Command Injection protection
const { execFile } = require('child_process');

// ✗ UNSAFE: never do this
app.post('/api/convert', (req, res) => {
  const filename = req.body.filename;
  exec(`convert ${filename} output.pdf`, (error, stdout) => {
    // Attacker can inject: filename = "input.jpg; rm -rf /"
  });
});

// ✓ SAFE: use execFile with array arguments
app.post('/api/convert', (req, res) => {
  const filename = req.body.filename;

  // Validate filename
  if (!/^[a-zA-Z0-9_-]+\.(jpg|png)$/.test(filename)) {
    return res.status(400).json({ error: 'Invalid filename' });
  }

  execFile('convert', [filename, 'output.pdf'], (error, stdout) => {
    if (error) {
      return res.status(500).json({ error: 'Conversion failed' });
    }
    res.json({ success: true });
  });
});
```

---

## 4. Encryption

### Encryption at Rest

```javascript
const crypto = require('crypto');

class EncryptionService {
  constructor(masterKey) {
    this.algorithm = 'aes-256-gcm';
    this.masterKey = Buffer.from(masterKey, 'hex');
  }

  encrypt(plaintext) {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv(this.algorithm, this.masterKey, iv);

    let encrypted = cipher.update(plaintext, 'utf8', 'hex');
    encrypted += cipher.final('hex');

    const authTag = cipher.getAuthTag();

    return {
      iv: iv.toString('hex'),
      encrypted,
      authTag: authTag.toString('hex')
    };
  }

  decrypt(iv, encrypted, authTag) {
    const decipher = crypto.createDecipheriv(
      this.algorithm,
      this.masterKey,
      Buffer.from(iv, 'hex')
    );

    decipher.setAuthTag(Buffer.from(authTag, 'hex'));

    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');

    return decrypted;
  }
}

// Usage: encrypt sensitive data before storing
const encryption = new EncryptionService(process.env.ENCRYPTION_KEY);

app.post('/api/credit-cards', async (req, res) => {
  const { cardNumber, cvv } = req.body;

  // Encrypt sensitive data
  const encryptedCard = encryption.encrypt(cardNumber);
  const encryptedCVV = encryption.encrypt(cvv);

  await db.query(
    `INSERT INTO credit_cards (user_id, card_number_encrypted, card_number_iv, card_number_tag, cvv_encrypted, cvv_iv, cvv_tag)
     VALUES ($1, $2, $3, $4, $5, $6, $7)`,
    [
      req.userId,
      encryptedCard.encrypted,
      encryptedCard.iv,
      encryptedCard.authTag,
      encryptedCVV.encrypted,
      encryptedCVV.iv,
      encryptedCVV.authTag
    ]
  );

  res.json({ success: true });
});

app.get('/api/credit-cards/:id', async (req, res) => {
  const result = await db.query(
    'SELECT * FROM credit_cards WHERE id = $1 AND user_id = $2',
    [req.params.id, req.userId]
  );

  const card = result.rows[0];

  // Decrypt
  const cardNumber = encryption.decrypt(
    card.card_number_iv,
    card.card_number_encrypted,
    card.card_number_tag
  );

  res.json({
    cardNumber: '****' + cardNumber.slice(-4),  // Only last 4 digits
    fullCardNumber: cardNumber  // Only for authorized users
  });
});
```

### Encryption in Transit (TLS)

```javascript
const https = require('https');
const fs = require('fs');

const options = {
  key: fs.readFileSync('/path/to/private-key.pem'),
  cert: fs.readFileSync('/path/to/certificate.pem'),
  // Modern TLS settings
  minVersion: 'TLSv1.2',
  ciphers: [
    'TLS_AES_128_GCM_SHA256',
    'TLS_AES_256_GCM_SHA384',
    'TLS_CHACHA20_POLY1305_SHA256',
    'ECDHE-RSA-AES128-GCM-SHA256',
    'ECDHE-RSA-AES256-GCM-SHA384'
  ].join(':'),
  honorCipherOrder: true
};

https.createServer(options, app).listen(443);
```

**Nginx TLS configuration:**

```nginx
server {
    listen 443 ssl http2;
    server_name api.example.com;

    # TLS certificates
    ssl_certificate /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;

    # Modern TLS configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers off;

    # HSTS (enforce HTTPS)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/api.example.com/chain.pem;

    # Session cache
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

---

## 5. Security Headers

```javascript
const helmet = require('helmet');

app.use(helmet({
  // Content Security Policy
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'", "https://cdn.example.com"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'", "https://api.example.com"],
      fontSrc: ["'self'", "https://fonts.gstatic.com"],
      objectSrc: ["'none'"],
      mediaSrc: ["'self'"],
      frameSrc: ["'none'"]
    }
  },

  // HSTS
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  },

  // X-Frame-Options (clickjacking protection)
  frameguard: {
    action: 'deny'
  },

  // X-Content-Type-Options
  noSniff: true,

  // Referrer-Policy
  referrerPolicy: {
    policy: 'strict-origin-when-cross-origin'
  }
}));

// CORS configuration
const cors = require('cors');

app.use(cors({
  origin: ['https://app.example.com', 'https://admin.example.com'],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

---

## 6. Rate Limiting and DDoS Protection

```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');

// Global rate limit
const globalLimiter = rateLimit({
  store: new RedisStore({
    client: redis,
    prefix: 'rl:global:'
  }),
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 100,                   // 100 requests per window
  message: 'Too many requests, please try again later',
  standardHeaders: true,
  legacyHeaders: false
});

// Strict rate limit for auth endpoints
const authLimiter = rateLimit({
  store: new RedisStore({
    client: redis,
    prefix: 'rl:auth:'
  }),
  windowMs: 15 * 60 * 1000,
  max: 5,  // Only 5 login attempts per 15 minutes
  skipSuccessfulRequests: true,
  message: 'Too many login attempts, please try again later'
});

app.use('/api/', globalLimiter);
app.use('/auth/', authLimiter);

// Advanced: per-user rate limiting
class UserRateLimiter {
  constructor(redis) {
    this.redis = redis;
  }

  async checkLimit(userId, action, maxRequests, windowSeconds) {
    const key = `rl:user:${userId}:${action}`;
    const now = Date.now();
    const windowStart = now - (windowSeconds * 1000);

    // Remove old entries
    await this.redis.zremrangebyscore(key, 0, windowStart);

    // Count requests in window
    const count = await this.redis.zcard(key);

    if (count >= maxRequests) {
      const ttl = await this.redis.pttl(key);
      throw new Error(`Rate limit exceeded. Try again in ${Math.ceil(ttl / 1000)}s`);
    }

    // Add current request
    await this.redis.zadd(key, now, `${now}-${Math.random()}`);
    await this.redis.expire(key, windowSeconds);

    return {
      allowed: true,
      remaining: maxRequests - count - 1
    };
  }
}

const userLimiter = new UserRateLimiter(redis);

app.post('/api/expensive-operation', async (req, res) => {
  try {
    await userLimiter.checkLimit(req.userId, 'expensive-op', 10, 3600);  // 10 per hour
    // ... perform operation
    res.json({ success: true });
  } catch (error) {
    res.status(429).json({ error: error.message });
  }
});
```

---

## 7. Secrets Management

### AWS Secrets Manager

```javascript
const { SecretsManagerClient, GetSecretValueCommand } = require('@aws-sdk/client-secrets-manager');

class SecretsService {
  constructor() {
    this.client = new SecretsManagerClient({ region: 'us-east-1' });
    this.cache = new Map();
    this.cacheTTL = 5 * 60 * 1000;  // 5 minutes
  }

  async getSecret(secretName) {
    // Check cache
    const cached = this.cache.get(secretName);
    if (cached && Date.now() - cached.timestamp < this.cacheTTL) {
      return cached.value;
    }

    // Fetch from Secrets Manager
    const command = new GetSecretValueCommand({ SecretId: secretName });
    const response = await this.client.send(command);

    const secret = JSON.parse(response.SecretString);

    // Cache
    this.cache.set(secretName, {
      value: secret,
      timestamp: Date.now()
    });

    return secret;
  }
}

const secrets = new SecretsService();

// Usage
async function initApp() {
  const dbCreds = await secrets.getSecret('production/database');

  const db = new Pool({
    host: dbCreds.host,
    user: dbCreds.username,
    password: dbCreds.password,
    database: dbCreds.database
  });
}
```

### Kubernetes Secrets with External Secrets Operator

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa

---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: db-secret
    creationPolicy: Owner
  data:
    - secretKey: DB_HOST
      remoteRef:
        key: production/database
        property: host
    - secretKey: DB_PASSWORD
      remoteRef:
        key: production/database
        property: password
```

---

## 8. Audit Logging

```javascript
class AuditLogger {
  constructor(db) {
    this.db = db;
  }

  async log(event) {
    await this.db.query(
      `INSERT INTO audit_logs (user_id, action, resource, resource_id, ip_address, user_agent, metadata, created_at)
       VALUES ($1, $2, $3, $4, $5, $6, $7, NOW())`,
      [
        event.userId,
        event.action,
        event.resource,
        event.resourceId,
        event.ipAddress,
        event.userAgent,
        JSON.stringify(event.metadata)
      ]
    );
  }
}

const auditLogger = new AuditLogger(db);

// Middleware
app.use((req, res, next) => {
  const originalSend = res.send;

  res.send = function(data) {
    // Log after response
    if (req.userId && res.statusCode < 400) {
      auditLogger.log({
        userId: req.userId,
        action: req.method,
        resource: req.path,
        resourceId: req.params.id,
        ipAddress: req.ip,
        userAgent: req.headers['user-agent'],
        metadata: {
          statusCode: res.statusCode,
          body: req.body
        }
      }).catch(console.error);
    }

    originalSend.call(this, data);
  };

  next();
});
```

---

## 9. Dependency Security

```bash
# Check for vulnerabilities
npm audit

# Fix automatically
npm audit fix

# Use Snyk for continuous monitoring
snyk test
snyk monitor

# Use Dependabot (GitHub)
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
```

---

## 10. Security Monitoring

```yaml
# Falco rules (Runtime security for Kubernetes)
- rule: Suspicious Network Activity
  desc: Detect suspicious network connections
  condition: outbound and not trusted_network
  output: "Suspicious network activity (user=%user.name command=%proc.cmdline connection=%fd.name)"
  priority: WARNING

- rule: Write Below Binary Dir
  desc: Detect attempts to write to binary directories
  condition: >
    evt.type = write and
    fd.name startswith /bin or fd.name startswith /usr/bin
  output: "Write below binary directory (user=%user.name command=%proc.cmdline file=%fd.name)"
  priority: CRITICAL
```

---

## Best Practices Summary

### 1. Principle of Least Privilege
Предоставляйте минимальные необходимые права.

### 2. Defense in Depth
Множество слоёв защиты.

### 3. Fail Securely
При ошибке — deny access, не grant.

### 4. Never Trust User Input
Всегда валидируйте и санитизируйте.

### 5. Keep Secrets Secret
Никогда не коммитьте secrets в Git.

### 6. Encrypt Everything
At rest и in transit.

### 7. Update Dependencies
Регулярно обновляйте зависимости.

### 8. Monitor and Alert
Быстрое обнаружение инцидентов критично.

### 9. Security by Design
Думайте о безопасности с первого дня.

### 10. Regular Security Audits
Penetration testing, code reviews.

---

## Проверь себя

1. Что такое CIA Triad (Confidentiality, Integrity, Availability)?
2. Как защититься от SQL Injection?
3. Что такое JWT и как правильно его использовать?
4. В чём разница между Authentication и Authorization?
5. Зачем нужен HTTPS/TLS?
6. Как защититься от DDoS атак?
7. Что такое RBAC (Role-Based Access Control)?
8. Зачем нужны Security Headers (CSP, HSTS, X-Frame-Options)?
9. Как безопасно хранить пароли (bcrypt, Argon2)?
10. Что такое Secrets Management и почему нельзя хранить secrets в коде?

---

[← Урок 56: Disaster Recovery](./56-disaster-recovery.md) | [README →](../README.md)

---

## Поздравляем! 🎉

Вы прошли все 57 уроков курса по System Design!

Теперь вы знаете:
- ✓ Фундаментальные концепции (latency, throughput, CAP theorem)
- ✓ Networking и коммуникации (HTTP, WebSocket, gRPC, GraphQL)
- ✓ Базы данных (SQL, NoSQL, NewSQL, sharding, replication)
- ✓ Кэширование и очереди (Redis, Kafka, RabbitMQ)
- ✓ Микросервисы и паттерны (API Gateway, Circuit Breaker, CQRS, Saga)
- ✓ Distributed systems (Consensus, Leader Election, Vector Clocks)
- ✓ Реальные системы (URL Shortener, Chat, E-commerce, Netflix, Uber, Facebook)
- ✓ Production (Monitoring, Deployment, Kubernetes, DR, Security)

**Следующие шаги:**
1. Практикуйтесь: реализуйте проекты из курса
2. Читайте документацию реальных систем (AWS, Netflix, Uber tech blogs)
3. Проходите mock interviews на system design
4. Вносите вклад в open source проекты

Удачи в ваших интервью и проектах! 🚀
