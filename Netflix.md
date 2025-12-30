# Netflix - System Design


<img width="5525" height="7958" alt="Netflix(OTT)" src="https://github.com/user-attachments/assets/873c7621-c230-44fd-9eac-9aa220a0eece" />

## 1. Functional Requirements (FR)

### Core Features
1. **User Management**
   - User registration and authentication
   - Multiple profiles per account
   - User preferences and settings
   - Parental controls

2. **Content Browsing**
   - Browse movies and TV shows by categories
   - Search for content
   - Personalized recommendations
   - Continue watching
   - My list / Watchlist

3. **Video Streaming**
   - Play video content
   - Adaptive bitrate streaming (ABR)
   - Multiple quality options (480p, 720p, 1080p, 4K)
   - Resume playback from last position
   - Subtitles and audio tracks selection
   - Skip intro/credits

4. **Content Management**
   - Upload and encode videos
   - Store metadata (title, description, cast, genre)
   - Thumbnail generation
   - Preview generation (short clips)

5. **Social Features**
   - Ratings and reviews
   - Share content
   - User activity tracking (for recommendations)

### Out of Scope
- Content production and licensing
- Payment processing (assume integrated)
- Live streaming
- User-generated content

## 2. Non-Functional Requirements (NFR)

1. **Scalability**
   - Support 200M+ subscribers globally
   - Handle 100M+ concurrent viewers during peak hours
   - Serve content to any location worldwide
   - Handle traffic spikes during popular releases

2. **Performance**
   - Video startup time: < 1 second
   - Search results: < 500ms
   - Home page load: < 1 second
   - 99th percentile latency: < 2 seconds
   - Smooth playback with minimal buffering

3. **Availability**
   - 99.99% uptime
   - Graceful degradation (show cached content if services fail)
   - No single point of failure
   - Regional failover

4. **Bandwidth Optimization**
   - Adaptive bitrate streaming
   - Video compression
   - CDN for content delivery
   - Minimize bandwidth costs

5. **Consistency**
   - Eventual consistency for user preferences
   - Strong consistency for payment/subscription status
   - Watch history eventually consistent

6. **Content Delivery**
   - Low latency globally (< 100ms to CDN)
   - High throughput for 4K streaming
   - Efficient CDN utilization

## 3. Core Entities

### 3.1 User
```
User {
  userId: UUID
  email: string
  passwordHash: string
  subscriptionPlan: enum (BASIC, STANDARD, PREMIUM)
  subscriptionStatus: enum (ACTIVE, CANCELED, EXPIRED)
  subscriptionDate: timestamp
  country: string
  createdAt: timestamp
}
```

### 3.2 Profile
```
Profile {
  profileId: UUID
  userId: UUID (User)
  name: string
  avatarUrl: string
  isKids: boolean
  language: string
  preferences: JSON
  createdAt: timestamp
}
```

### 3.3 Content (Movie/Show)
```
Content {
  contentId: UUID
  title: string
  description: text
  type: enum (MOVIE, TV_SHOW)
  genres: List<string>
  releaseYear: int
  duration: int (minutes)
  rating: string (PG, PG-13, R, etc.)
  language: string
  cast: List<string>
  director: string
  thumbnailUrl: string
  trailerUrl: string
  createdAt: timestamp
  updatedAt: timestamp
}
```

### 3.4 Video
```
Video {
  videoId: UUID
  contentId: UUID (Content)
  seasonNumber: int (nullable)
  episodeNumber: int (nullable)
  title: string
  duration: int (seconds)
  videoFiles: List<VideoFile>
  subtitles: List<Subtitle>
  audioTracks: List<AudioTrack>
}
```

### 3.5 VideoFile
```
VideoFile {
  fileId: UUID
  videoId: UUID
  quality: enum (480p, 720p, 1080p, 4K)
  codec: string (H.264, H.265)
  bitrate: int (Mbps)
  resolution: string (1920x1080)
  storageLocation: string (S3/CDN path)
  size: long (bytes)
}
```

### 3.6 WatchHistory
```
WatchHistory {
  historyId: UUID
  profileId: UUID
  videoId: UUID
  watchedDuration: int (seconds)
  totalDuration: int (seconds)
  lastWatchedAt: timestamp
  completed: boolean
}
```

### 3.7 Recommendation
```
Recommendation {
  recommendationId: UUID
  profileId: UUID
  contentId: UUID
  score: float
  reason: string
  generatedAt: timestamp
}
```

### 3.8 UserRating
```
UserRating {
  ratingId: UUID
  profileId: UUID
  contentId: UUID
  rating: int (1-5 stars)
  createdAt: timestamp
}
```

### 3.9 MyList
```
MyList {
  listId: UUID
  profileId: UUID
  contentId: UUID
  addedAt: timestamp
}
```

## 4. API Design

### 4.1 User & Authentication APIs
```
POST /api/v1/auth/signup
Request: { email, password, country }
Response: { userId, token }

POST /api/v1/auth/login
Request: { email, password }
Response: { userId, token, profiles: [] }

POST /api/v1/profiles
Headers: { Authorization: Bearer <token> }
Request: { name, isKids, avatarUrl }
Response: { profileId, name }

GET /api/v1/profiles
Headers: { Authorization: Bearer <token> }
Response: {
  profiles: [
    { profileId, name, avatarUrl, isKids }
  ]
}
```

### 4.2 Content Browsing APIs
```
GET /api/v1/content/home
Headers: { Authorization: Bearer <token>, Profile-Id: <profileId> }
Response: {
  continueWatching: [...],
  trending: [...],
  recommendations: [...],
  categories: [
    {
      name: "Action",
      content: [...]
    }
  ]
}

GET /api/v1/content/search
Headers: { Authorization: Bearer <token> }
Query: { query: string, page: int, limit: int }
Response: {
  results: [
    {
      contentId, title, thumbnailUrl, type, rating
    }
  ],
  totalCount: int
}

GET /api/v1/content/{contentId}
Headers: { Authorization: Bearer <token> }
Response: {
  contentId, title, description, genres, releaseYear,
  rating, cast, director, thumbnailUrl, trailerUrl,
  episodes: [ ... ] (if TV show)
}

GET /api/v1/content/category/{category}
Headers: { Authorization: Bearer <token> }
Query: { page, limit, sortBy }
Response: {
  content: [...],
  totalCount: int
}
```

### 4.3 Video Streaming APIs
```
GET /api/v1/video/{videoId}/play
Headers: { Authorization: Bearer <token>, Profile-Id: <profileId> }
Response: {
  videoId,
  manifestUrl: string (HLS/DASH manifest),
  qualities: [
    { quality: "1080p", url: string, bitrate: int }
  ],
  subtitles: [
    { language: "en", url: string }
  ],
  audioTracks: [
    { language: "en", url: string }
  ],
  resumePosition: int (seconds)
}

POST /api/v1/video/{videoId}/progress
Headers: { Authorization: Bearer <token>, Profile-Id: <profileId> }
Request: { position: int (seconds), duration: int }
Response: { success: true }

GET /api/v1/video/{videoId}/manifest.m3u8
Headers: { Authorization: Bearer <token> }
Response: HLS manifest file
```

### 4.4 User Activity APIs
```
GET /api/v1/profile/{profileId}/watch-history
Headers: { Authorization: Bearer <token> }
Query: { page, limit }
Response: {
  history: [
    {
      videoId, contentId, title, thumbnailUrl,
      progress: float (0-1), lastWatchedAt
    }
  ]
}

POST /api/v1/profile/{profileId}/my-list
Headers: { Authorization: Bearer <token> }
Request: { contentId }
Response: { success: true }

DELETE /api/v1/profile/{profileId}/my-list/{contentId}
Headers: { Authorization: Bearer <token> }
Response: { success: true }

GET /api/v1/profile/{profileId}/my-list
Headers: { Authorization: Bearer <token> }
Response: {
  content: [
    { contentId, title, thumbnailUrl, addedAt }
  ]
}
```

### 4.5 Rating & Review APIs
```
POST /api/v1/content/{contentId}/rating
Headers: { Authorization: Bearer <token>, Profile-Id: <profileId> }
Request: { rating: int (1-5) }
Response: { success: true, averageRating: float }

GET /api/v1/content/{contentId}/rating
Headers: { Authorization: Bearer <token> }
Response: {
  averageRating: float,
  totalRatings: int,
  userRating: int (if exists)
}
```

### 4.6 Recommendation API
```
GET /api/v1/profile/{profileId}/recommendations
Headers: { Authorization: Bearer <token> }
Query: { page, limit }
Response: {
  recommendations: [
    {
      contentId, title, thumbnailUrl, score,
      reason: "Because you watched X"
    }
  ]
}
```

### 4.7 Admin APIs (Content Upload)
```
POST /api/v1/admin/content
Headers: { Authorization: Bearer <admin-token> }
Request: {
  title, description, type, genres, releaseYear,
  rating, language, cast, director
}
Response: { contentId }

POST /api/v1/admin/content/{contentId}/video
Headers: { Authorization: Bearer <admin-token> }
Request: multipart/form-data
  - video: binary
  - seasonNumber: int (optional)
  - episodeNumber: int (optional)
Response: {
  videoId,
  encodingJobId: UUID
}

GET /api/v1/admin/encoding/{jobId}/status
Headers: { Authorization: Bearer <admin-token> }
Response: {
  jobId,
  status: enum (PENDING, PROCESSING, COMPLETED, FAILED),
  progress: int (0-100)
}
```

## 5. High-Level Design (HLD)

### 5.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                       CLIENT LAYER                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Web App     │  │  Mobile App  │  │   TV App     │          │
│  │  (React)     │  │ (iOS/Android)│  │  (SmartTV)   │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
└─────────┼──────────────────┼──────────────────┼──────────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
                ┌────────────▼─────────────┐
                │   CDN (CloudFront/       │
                │   Akamai/Fastly)         │
                │   - Video Content        │
                │   - Images/Thumbnails    │
                └────────────┬─────────────┘
                             │
                ┌────────────▼─────────────┐
                │   Load Balancer          │
                │   (AWS ALB / NGINX)      │
                └────────────┬─────────────┘
                             │
┌────────────────────────────┼──────────────────────────────────┐
│                    API GATEWAY LAYER                           │
│                ┌───────────▼─────────────┐                     │
│                │   API Gateway           │                     │
│                │   - Authentication      │                     │
│                │   - Rate Limiting       │                     │
│                │   - Request Routing     │                     │
│                └───────────┬─────────────┘                     │
└────────────────────────────┼──────────────────────────────────┘
                             │
┌────────────────────────────┼──────────────────────────────────┐
│              MICROSERVICES LAYER                               │
│                             │                                  │
│    ┌────────────┬──────────┼──────────┬──────────┐            │
│    │            │          │          │          │            │
│ ┌──▼──────┐ ┌──▼────┐ ┌───▼────┐ ┌───▼────┐ ┌──▼────────┐    │
│ │  User   │ │Content│ │Streaming│ │Search  │ │Recommend  │    │
│ │ Service │ │Service│ │ Service │ │Service │ │  Service  │    │
│ └──┬──────┘ └──┬────┘ └───┬────┘ └───┬────┘ └──┬────────┘    │
│    │           │          │          │          │            │
│ ┌──▼──────┐ ┌──▼────┐ ┌───▼────┐ ┌───▼────┐ ┌──▼────────┐    │
│ │ Watch   │ │Rating │ │ Video  │ │ Notif. │ │ Analytics │    │
│ │ History │ │Service│ │Encoding│ │Service │ │  Service  │    │
│ │ Service │ │       │ │Service │ │        │ │           │    │
│ └──┬──────┘ └──┬────┘ └───┬────┘ └───┬────┘ └──┬────────┘    │
└────┼───────────┼──────────┼──────────┼──────────┼────────────┘
     │           │          │          │          │
┌────┼───────────┼──────────┼──────────┼──────────┼────────────┐
│    │      DATA & STORAGE LAYER        │          │            │
│ ┌──▼──────┐ ┌──▼────────┐ ┌───▼──────▼──┐ ┌─────▼──────┐     │
│ │PostgreSQL│ │ Cassandra │ │    Redis    │ │ElasticSearch│    │
│ │(User,    │ │(Watch     │ │  (Cache,    │ │  (Content   │    │
│ │ Content  │ │ History,  │ │  Sessions)  │ │   Search)   │    │
│ │Metadata) │ │ Ratings)  │ │             │ │             │    │
│ └──────────┘ └───────────┘ └─────────────┘ └─────────────┘    │
│ ┌──────────────────────┐  ┌─────────────┐  ┌──────────────┐   │
│ │  S3 / Object Storage │  │   Kafka     │  │   Redshift   │   │
│ │  (Video Files,       │  │  (Event     │  │  (Analytics/ │   │
│ │   Thumbnails)        │  │   Stream)   │  │   Data WH)   │   │
│ └──────────────────────┘  └─────────────┘  └──────────────┘   │
└───────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────┐
│                  VIDEO ENCODING PIPELINE                       │
│  ┌────────────┐   ┌─────────────┐   ┌──────────────┐         │
│  │ Upload     │──>│  Encoding   │──>│  CDN Push    │         │
│  │ Service    │   │  Worker     │   │  Service     │         │
│  └────────────┘   └─────────────┘   └──────────────┘         │
└───────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────┐
│                ML/RECOMMENDATION PIPELINE                      │
│  ┌────────────┐   ┌─────────────┐   ┌──────────────┐         │
│  │  Data      │──>│ ML Training │──>│ Model Serving│         │
│  │ Collection │   │  Pipeline   │   │  (TensorFlow)│         │
│  └────────────┘   └─────────────┘   └──────────────┘         │
└───────────────────────────────────────────────────────────────┘
```

### 5.2 Component Descriptions

**Client Layer:**
- Web, mobile, and TV applications
- Adaptive bitrate player (Video.js, ExoPlayer)
- Offline download capability (mobile)

**CDN Layer:**
- Distributed globally (multi-region)
- Caches video files, thumbnails, static assets
- Reduces latency and bandwidth costs
- Points of Presence (PoP) near users

**API Gateway:**
- Authentication/Authorization (JWT tokens)
- Rate limiting (per user/IP)
- Request routing to microservices
- SSL termination

**Microservices:**
- **User Service**: User accounts, profiles, subscriptions
- **Content Service**: Content metadata, CRUD operations
- **Streaming Service**: Video delivery, manifest generation
- **Search Service**: Full-text search on content
- **Recommendation Service**: Personalized recommendations using ML
- **Watch History Service**: Track viewing progress
- **Rating Service**: User ratings and reviews
- **Video Encoding Service**: Transcode videos to multiple formats
- **Notification Service**: Push notifications, emails
- **Analytics Service**: User behavior analytics

**Data Layer:**
- **PostgreSQL**: User data, content metadata
- **Cassandra**: Watch history, ratings (high write volume)
- **Redis**: Session cache, frequently accessed data
- **ElasticSearch**: Content search indexing
- **S3**: Video files, thumbnails, assets
- **Kafka**: Event streaming for analytics
- **Redshift**: Data warehouse for analytics

### 5.3 Video Streaming Flow

```
┌─────────┐         ┌──────────┐        ┌─────────┐        ┌────────┐
│ Client  │         │Streaming │        │  CDN    │        │   S3   │
│         │         │ Service  │        │         │        │        │
└────┬────┘         └────┬─────┘        └────┬────┘        └────┬───┘
     │                   │                   │                  │
     │ Request Video     │                   │                  │
     ├──────────────────>│                   │                  │
     │                   │ Check Auth        │                  │
     │                   ├──────────┐        │                  │
     │                   │          │        │                  │
     │                   │<─────────┘        │                  │
     │                   │                   │                  │
     │                   │ Get Resume Pos    │                  │
     │                   ├──────────┐        │                  │
     │                   │          │        │                  │
     │                   │<─────────┘        │                  │
     │                   │                   │                  │
     │ Manifest URL +    │                   │                  │
     │ Resume Position   │                   │                  │
     │<──────────────────┤                   │                  │
     │                   │                   │                  │
     │ Request Manifest (HLS/DASH)           │                  │
     ├───────────────────┼──────────────────>│                  │
     │                   │                   │ Cache Miss?      │
     │                   │                   ├─────────────────>│
     │                   │                   │ Fetch Manifest   │
     │                   │                   │<─────────────────┤
     │                   │                   │                  │
     │ Manifest File     │                   │                  │
     │<──────────────────┼───────────────────┤                  │
     │                   │                   │                  │
     │ Request Video Segments (adaptive)     │                  │
     ├───────────────────┼──────────────────>│                  │
     │                   │                   │ Fetch Segments   │
     │                   │                   ├─────────────────>│
     │                   │                   │<─────────────────┤
     │ Video Segments    │                   │                  │
     │<──────────────────┼───────────────────┤                  │
     │                   │                   │                  │
     │ Report Progress   │                   │                  │
     ├──────────────────>│                   │                  │
     │                   │ Update Watch Hist │                  │
     │                   ├──────────┐        │                  │
     │                   │          │        │                  │
     │                   │<─────────┘        │                  │
```

### 5.4 Video Encoding Pipeline

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  Admin Upload         Encoding Queue         Encoding Worker  │
│       │                     │                      │           │
│       │  Upload Video       │                      │           │
│       ├────────────────────>│                      │           │
│       │                     │                      │           │
│       │                     │  Pick Job            │           │
│       │                     ├─────────────────────>│           │
│       │                     │                      │           │
│       │                     │                   Transcode      │
│       │                     │                   - 480p (H.264) │
│       │                     │                   - 720p (H.264) │
│       │                     │                   - 1080p(H.264) │
│       │                     │                   - 4K   (H.265) │
│       │                     │                      │           │
│       │                     │                   Generate       │
│       │                     │                   - Thumbnails   │
│       │                     │                   - Preview      │
│       │                     │                   - HLS Manifest │
│       │                     │                      │           │
│       │                     │                   Upload to S3   │
│       │                     │                   & CDN          │
│       │                     │                      │           │
│       │                     │  Job Complete        │           │
│       │  Notify Complete    │<─────────────────────┤           │
│       │<────────────────────┤                      │           │
│       │                     │                      │           │
└────────────────────────────────────────────────────────────────┘
```

### 5.5 Recommendation System

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  User Activity    ──>  Data Collection  ──>  Feature Store    │
│  (Watch, Rate,         (Kafka Stream)       (User features,   │
│   Search, etc.)                              Content features) │
│                                                   │            │
│                                                   │            │
│                                                   ▼            │
│                                            ML Training         │
│                                            (Spark/TensorFlow)  │
│                                            - Collaborative     │
│                                              Filtering         │
│                                            - Content-based     │
│                                            - Deep Learning     │
│                                                   │            │
│                                                   │            │
│                                                   ▼            │
│                                            Model Registry      │
│                                            (Trained Models)    │
│                                                   │            │
│                                                   │            │
│                                                   ▼            │
│  Client Request  ──>  Recommendation   <───  Model Serving    │
│                       Service                (TensorFlow       │
│                                               Serving)         │
│                          │                                     │
│                          │                                     │
│                          ▼                                     │
│                     Personalized                               │
│                     Results                                    │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 6. Optimizations & Approaches

### 6.1 Adaptive Bitrate Streaming (ABR)
**Problem**: Different network conditions require different video qualities
**Solution**: HLS (HTTP Live Streaming) or DASH
- Encode video in multiple bitrates
- Client measures bandwidth and switches quality dynamically
- Smooth playback without buffering

**Bitrate Ladder:**
```
4K:    25 Mbps (3840x2160) - H.265
1080p: 8 Mbps  (1920x1080) - H.264
720p:  5 Mbps  (1280x720)  - H.264
480p:  2 Mbps  (854x480)   - H.264
360p:  1 Mbps  (640x360)   - H.264
```

### 6.2 CDN Strategy
**Problem**: Serving video from origin is expensive and slow
**Solution**: Multi-tier CDN
- **Edge Servers**: Close to users (100+ locations globally)
- **Mid-tier Servers**: Regional caches
- **Origin**: S3 or object storage

**Open Connect (Netflix's own CDN)**:
- Netflix deploys servers directly in ISP networks
- Reduces transit costs
- Improves performance
- Cache popular content close to users

### 6.3 Predictive Caching
**Problem**: Don't want to cache all content everywhere (expensive)
**Solution**: Intelligent cache placement
- Predict what users will watch based on trends
- Pre-cache popular content in each region
- Time-based caching (new releases, trending shows)
- Cache eviction based on popularity decay

### 6.4 Video Compression
**Technology**:
- H.264 (AVC) for compatibility
- H.265 (HEVC) for 4K (50% better compression)
- AV1 for next-gen (30% better than H.265, royalty-free)

**Encoding Settings:**
- Variable bitrate (VBR) for better quality
- 2-pass encoding for optimal compression
- Per-title encoding (optimize bitrate for each content)

### 6.5 Database Sharding

**User Service (PostgreSQL):**
- Shard by userId (consistent hashing)
- Each shard: 1-10M users

**Watch History (Cassandra):**
- Partition by (profileId, videoId)
- Time-series data optimized
- TTL for old data

**Content Metadata (PostgreSQL):**
- Relatively small dataset (millions of titles)
- Read replicas for scaling reads
- Cache aggressively in Redis

### 6.6 Caching Strategy

**Multi-Level Cache:**
1. **Client Cache**: Downloaded content, metadata
2. **CDN Cache**: Video files, thumbnails
3. **Application Cache (Redis)**:
   - User session: 30 minutes
   - Content metadata: 1 hour
   - Home page recommendations: 10 minutes
   - Search results: 5 minutes

**Cache Invalidation:**
- Time-based (TTL)
- Event-driven (new content published)
- Write-through for user preferences

### 6.7 Search Optimization
**ElasticSearch Index:**
- Index all content with title, description, cast, genres
- Fuzzy matching for typos
- Boosting (title > description)
- Faceted search (filter by genre, year, rating)
- Auto-complete suggestions

**Example Query:**
```json
{
  "query": {
    "multi_match": {
      "query": "action comedy",
      "fields": ["title^3", "description", "genres^2"],
      "fuzziness": "AUTO"
    }
  },
  "filter": {
    "term": { "rating": "PG-13" }
  }
}
```

### 6.8 Recommendation Algorithm

**Collaborative Filtering:**
- User-based: Find similar users, recommend what they watched
- Item-based: Find similar content based on co-watching patterns
- Matrix factorization (SVD, ALS)

**Content-Based Filtering:**
- Recommend based on genres, cast, director
- TF-IDF on content metadata
- Similar to what user already watched

**Deep Learning:**
- Neural Collaborative Filtering
- Deep & Wide networks
- RNNs for sequence modeling (watch patterns)

**Hybrid Approach:**
- Combine multiple algorithms
- Contextual bandits for exploration/exploitation
- A/B testing for algorithm improvements

### 6.9 Thumbnail Personalization
**Netflix Innovation:**
- Generate multiple thumbnails per content
- Use ML to select best thumbnail for each user
- Based on user's watch history and preferences
- Increases click-through rate by 20-30%

### 6.10 Pre-fetching Strategy
**Problem**: Reduce startup latency
**Solution**:
- Pre-fetch first 30 seconds of video when user hovers over thumbnail
- Pre-load thumbnails for viewport
- Predict next episode, pre-fetch in background

### 6.11 Analytics & Monitoring
**Metrics to Track:**
- Video start time
- Buffering ratio
- Quality switches
- Abandonment rate
- Watch completion rate
- Search CTR

**Tools:**
- Real-time dashboards (Grafana)
- Distributed tracing (Jaeger)
- Log aggregation (ELK stack)
- Anomaly detection

## 7. Low-Level Design (LLD)

<img width="1798" height="1197" alt="image" src="https://github.com/user-attachments/assets/35649efc-405b-4853-869c-521ba3b4b28d" />

### 7.3 Database Schema

```sql
-- Users table
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    country VARCHAR(2) NOT NULL,
    subscription_plan VARCHAR(20) NOT NULL,
    subscription_status VARCHAR(20) NOT NULL,
    subscription_date TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    INDEX idx_email (email)
);

-- Profiles table
CREATE TABLE profiles (
    profile_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    name VARCHAR(100) NOT NULL,
    avatar_url VARCHAR(500),
    is_kids BOOLEAN NOT NULL DEFAULT FALSE,
    language VARCHAR(10) NOT NULL DEFAULT 'en',
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    INDEX idx_user (user_id)
);

-- Content table
CREATE TABLE content (
    content_id UUID PRIMARY KEY,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    type VARCHAR(20) NOT NULL, -- MOVIE, TV_SHOW
    release_year INT,
    rating VARCHAR(10),
    language VARCHAR(10),
    director VARCHAR(200),
    thumbnail_url VARCHAR(500),
    trailer_url VARCHAR(500),
    view_count BIGINT NOT NULL DEFAULT 0,
    average_rating FLOAT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    INDEX idx_type (type),
    INDEX idx_release_year (release_year),
    FULLTEXT INDEX idx_title_desc (title, description)
);

-- Content genres (many-to-many)
CREATE TABLE content_genres (
    content_id UUID NOT NULL,
    genre VARCHAR(50) NOT NULL,
    PRIMARY KEY (content_id, genre),
    FOREIGN KEY (content_id) REFERENCES content(content_id) ON DELETE CASCADE,
    INDEX idx_genre (genre)
);

-- Content cast (many-to-many)
CREATE TABLE content_cast (
    content_id UUID NOT NULL,
    actor_name VARCHAR(200) NOT NULL,
    role VARCHAR(200),
    PRIMARY KEY (content_id, actor_name),
    FOREIGN KEY (content_id) REFERENCES content(content_id) ON DELETE CASCADE,
    INDEX idx_actor (actor_name)
);

-- Videos table
CREATE TABLE videos (
    video_id UUID PRIMARY KEY,
    content_id UUID NOT NULL,
    season_number INT,
    episode_number INT,
    title VARCHAR(500),
    duration INT NOT NULL, -- seconds
    manifest_url VARCHAR(500) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    FOREIGN KEY (content_id) REFERENCES content(content_id) ON DELETE CASCADE,
    INDEX idx_content (content_id)
);

-- Video files (different qualities)
CREATE TABLE video_files (
    file_id UUID PRIMARY KEY,
    video_id UUID NOT NULL,
    quality VARCHAR(20) NOT NULL,
    resolution VARCHAR(20) NOT NULL,
    codec VARCHAR(20) NOT NULL,
    bitrate INT NOT NULL,
    storage_location VARCHAR(500) NOT NULL,
    file_size BIGINT NOT NULL,
    FOREIGN KEY (video_id) REFERENCES videos(video_id) ON DELETE CASCADE,
    INDEX idx_video_quality (video_id, quality)
);

-- Cassandra: Watch history (high write volume)
CREATE TABLE watch_history (
    profile_id UUID,
    video_id UUID,
    content_id UUID,
    watched_duration INT,
    total_duration INT,
    last_watched_at TIMESTAMP,
    completed BOOLEAN,
    PRIMARY KEY (profile_id, video_id)
) WITH CLUSTERING ORDER BY (video_id DESC);

-- Cassandra: User ratings
CREATE TABLE user_ratings (
    profile_id UUID,
    content_id UUID,
    rating INT,
    created_at TIMESTAMP,
    PRIMARY KEY (profile_id, content_id)
);

-- My List (PostgreSQL)
CREATE TABLE my_list (
    list_id UUID PRIMARY KEY,
    profile_id UUID NOT NULL,
    content_id UUID NOT NULL,
    added_at TIMESTAMP NOT NULL DEFAULT NOW(),
    FOREIGN KEY (profile_id) REFERENCES profiles(profile_id) ON DELETE CASCADE,
    FOREIGN KEY (content_id) REFERENCES content(content_id) ON DELETE CASCADE,
    UNIQUE KEY uk_profile_content (profile_id, content_id),
    INDEX idx_profile (profile_id)
);
```

### 7.4 Scalability Calculations

**User Scale:**
- 200M subscribers
- 100M daily active users
- 50M concurrent peak users
- 10M concurrent streams

**Data Volume:**
- Content library: 10,000 movies + 5,000 TV shows (50,000 episodes)
- Average movie: 2 hours, 5 qualities = 50GB per movie
- Total: 10,000 × 50GB = 500TB of video content
- With redundancy and CDN: 5PB

**Bandwidth:**
- 10M concurrent streams × 8 Mbps (1080p) = 80 Tbps
- Average: 40 Tbps
- Daily: 40 Tbps × 86400s / 8 / 1024^4 ≈ 400 PB/day

**Database:**
- User data: 200M users × 1KB = 200GB
- Content metadata: 10K titles × 100KB = 1GB
- Watch history: 200M users × 1000 entries × 100 bytes = 20TB (Cassandra)

**Recommendations:**
- ML model inference: 100M recommendations/day
- Feature extraction: Real-time from watch history
- Model retraining: Daily batch job

---

## Interview Discussion Points

### Clarifying Questions:
1. Scale: How many users? Concurrent streams?
2. Content: Size of library? Upload frequency?
3. Quality: Max resolution? Bitrates?
4. Regions: Global or specific regions?
5. Features: Live streaming? Downloads?
6. Cost: Budget constraints for bandwidth/storage?

### Trade-offs:
1. **CDN vs Origin**: Cost vs performance
2. **Video Quality**: Storage/bandwidth vs user experience
3. **Recommendation**: Simple (fast) vs complex ML (accurate)
4. **Consistency**: Strong vs eventual for watch history

### Extensions:
1. **Live Streaming**: Sports, events
2. **Downloads**: Offline viewing
3. **Interactive Content**: Choose-your-own-adventure
4. **Social Features**: Watch parties, comments
5. **A/B Testing**: UI experimentation

This design provides a comprehensive foundation for a video streaming platform like Netflix at global scale!

