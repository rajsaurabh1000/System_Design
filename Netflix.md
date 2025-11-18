# Netflix - System Design

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

### 7.1 Class Diagram

```
┌──────────────────────────────────────────────────────────┐
│                      User                                │
├──────────────────────────────────────────────────────────┤
│ - userId: UUID                                           │
│ - email: string                                          │
│ - passwordHash: string                                   │
│ - subscription: Subscription                             │
│ - profiles: List<Profile>                                │
├──────────────────────────────────────────────────────────┤
│ + authenticate(password: string): boolean                │
│ + addProfile(profile: Profile): void                     │
│ + getActiveProfiles(): List<Profile>                     │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                     Profile                              │
├──────────────────────────────────────────────────────────┤
│ - profileId: UUID                                        │
│ - name: string                                           │
│ - isKids: boolean                                        │
│ - watchHistory: WatchHistory                             │
│ - myList: List<Content>                                  │
│ - preferences: UserPreferences                           │
├──────────────────────────────────────────────────────────┤
│ + addToWatchHistory(video: Video, position: int): void   │
│ + addToMyList(content: Content): void                    │
│ + getRecommendations(): List<Content>                    │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                     Content                              │
├──────────────────────────────────────────────────────────┤
│ - contentId: UUID                                        │
│ - title: string                                          │
│ - description: string                                    │
│ - type: ContentType (MOVIE, TV_SHOW)                     │
│ - genres: List<Genre>                                    │
│ - rating: ContentRating                                  │
│ - videos: List<Video>                                    │
│ - metadata: ContentMetadata                              │
├──────────────────────────────────────────────────────────┤
│ + getVideoById(videoId: UUID): Video                     │
│ + getThumbnail(userId: UUID): string                     │
│ + getTrailer(): Video                                    │
│ + getAverageRating(): float                              │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                      Video                               │
├──────────────────────────────────────────────────────────┤
│ - videoId: UUID                                          │
│ - contentId: UUID                                        │
│ - title: string                                          │
│ - duration: int (seconds)                                │
│ - videoFiles: List<VideoFile>                            │
│ - subtitles: List<Subtitle>                              │
│ - audioTracks: List<AudioTrack>                          │
├──────────────────────────────────────────────────────────┤
│ + getManifestUrl(): string                               │
│ + getQualityOptions(): List<Quality>                     │
│ + getSubtitle(language: string): Subtitle                │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                   StreamingService                       │
├──────────────────────────────────────────────────────────┤
│ - videoRepository: VideoRepository                       │
│ - cdnManager: CDNManager                                 │
│ - watchHistoryService: WatchHistoryService               │
├──────────────────────────────────────────────────────────┤
│ + getVideoManifest(videoId, quality): Manifest           │
│ + generateStreamingUrl(videoId, quality): string         │
│ + recordProgress(profileId, videoId, position): void     │
│ + getResumePosition(profileId, videoId): int             │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│               RecommendationService                      │
├──────────────────────────────────────────────────────────┤
│ - mlModel: MLModel                                       │
│ - userPreferenceService: UserPreferenceService           │
│ - contentService: ContentService                         │
├──────────────────────────────────────────────────────────┤
│ + getPersonalizedRecommendations(profileId): List<...>  │
│ + getTrendingContent(): List<Content>                    │
│ + getSimilarContent(contentId): List<Content>            │
│ + updateModel(): void                                    │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                  EncodingService                         │
├──────────────────────────────────────────────────────────┤
│ - encodingQueue: Queue<EncodingJob>                      │
│ - s3Client: S3Client                                     │
│ - cdnClient: CDNClient                                   │
├──────────────────────────────────────────────────────────┤
│ + submitEncodingJob(videoFile): UUID                     │
│ + encodeVideo(job: EncodingJob): List<VideoFile>         │
│ + generateThumbnails(videoFile): List<string>            │
│ + generateManifest(videoFiles): string                   │
│ + uploadToCDN(files: List<File>): void                   │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                   SearchService                          │
├──────────────────────────────────────────────────────────┤
│ - elasticSearchClient: ESClient                          │
│ - contentService: ContentService                         │
├──────────────────────────────────────────────────────────┤
│ + searchContent(query: string, filters): List<Content>   │
│ + indexContent(content: Content): void                   │
│ + getSuggestions(prefix: string): List<string>           │
│ + updateIndex(): void                                    │
└──────────────────────────────────────────────────────────┘
```

### 7.2 Detailed Component Implementation

#### 7.2.1 Streaming Service

```java
public class StreamingService {
    private final VideoRepository videoRepository;
    private final CDNManager cdnManager;
    private final WatchHistoryService watchHistoryService;
    private final AuthService authService;
    private final CacheManager cacheManager;
    
    public StreamingResponse getVideoStream(
        UUID videoId,
        UUID profileId,
        String authToken
    ) {
        // 1. Authenticate user
        User user = authService.validateToken(authToken);
        Profile profile = user.getProfile(profileId);
        
        if (profile == null) {
            throw new UnauthorizedException("Invalid profile");
        }
        
        // 2. Check subscription status
        if (!user.hasActiveSubscription()) {
            throw new SubscriptionExpiredException();
        }
        
        // 3. Get video details
        Video video = videoRepository.findById(videoId)
            .orElseThrow(() -> new VideoNotFoundException(videoId));
        
        // 4. Check content rating vs profile restrictions
        if (profile.isKids() && !video.isKidFriendly()) {
            throw new ContentRestrictedException("Not suitable for kids profile");
        }
        
        // 5. Get user's subscription tier for quality options
        List<VideoFile> availableQualities = getAvailableQualities(
            video,
            user.getSubscriptionPlan()
        );
        
        // 6. Generate HLS manifest URL
        String manifestUrl = cdnManager.generateManifestUrl(
            videoId,
            user.getCountry()
        );
        
        // 7. Get resume position from watch history
        int resumePosition = watchHistoryService.getResumePosition(
            profileId,
            videoId
        );
        
        // 8. Generate streaming URLs (signed with expiration)
        List<QualityOption> qualityOptions = new ArrayList<>();
        for (VideoFile vf : availableQualities) {
            String url = cdnManager.generateSignedUrl(
                vf.getStorageLocation(),
                Duration.ofHours(4)
            );
            qualityOptions.add(new QualityOption(
                vf.getQuality(),
                url,
                vf.getBitrate()
            ));
        }
        
        // 9. Get subtitles and audio tracks
        List<SubtitleOption> subtitles = video.getSubtitles().stream()
            .map(s -> new SubtitleOption(
                s.getLanguage(),
                cdnManager.generateSignedUrl(s.getUrl(), Duration.ofHours(4))
            ))
            .collect(Collectors.toList());
        
        List<AudioTrackOption> audioTracks = video.getAudioTracks().stream()
            .map(a -> new AudioTrackOption(
                a.getLanguage(),
                cdnManager.generateSignedUrl(a.getUrl(), Duration.ofHours(4))
            ))
            .collect(Collectors.toList());
        
        // 10. Log streaming event for analytics
        analyticsService.logEvent(new StreamingEvent(
            user.getUserId(),
            profileId,
            videoId,
            Instant.now()
        ));
        
        return StreamingResponse.builder()
            .videoId(videoId)
            .manifestUrl(manifestUrl)
            .qualityOptions(qualityOptions)
            .subtitles(subtitles)
            .audioTracks(audioTracks)
            .resumePosition(resumePosition)
            .duration(video.getDuration())
            .build();
    }
    
    private List<VideoFile> getAvailableQualities(
        Video video,
        SubscriptionPlan plan
    ) {
        List<VideoFile> allFiles = video.getVideoFiles();
        
        switch (plan) {
            case BASIC:
                // 480p max
                return allFiles.stream()
                    .filter(vf -> vf.getResolutionHeight() <= 480)
                    .collect(Collectors.toList());
            case STANDARD:
                // 1080p max
                return allFiles.stream()
                    .filter(vf -> vf.getResolutionHeight() <= 1080)
                    .collect(Collectors.toList());
            case PREMIUM:
                // 4K available
                return allFiles;
            default:
                return allFiles.stream()
                    .filter(vf -> vf.getResolutionHeight() <= 480)
                    .collect(Collectors.toList());
        }
    }
    
    public void updateWatchProgress(
        UUID profileId,
        UUID videoId,
        int position,
        int duration
    ) {
        // Debounce - only save every 10 seconds
        String cacheKey = String.format("progress:%s:%s", profileId, videoId);
        Long lastSave = cacheManager.get(cacheKey);
        
        if (lastSave != null && 
            System.currentTimeMillis() - lastSave < 10000) {
            // Too soon, skip
            return;
        }
        
        // Save to database
        watchHistoryService.updateProgress(
            profileId,
            videoId,
            position,
            duration,
            position >= duration * 0.9 // completed if 90% watched
        );
        
        // Update cache
        cacheManager.set(cacheKey, System.currentTimeMillis(), Duration.ofMinutes(1));
    }
}
```

#### 7.2.2 Video Encoding Service

```java
public class EncodingService {
    private final S3Client s3Client;
    private final SQSClient sqsClient;
    private final DynamoDBClient dynamoClient;
    
    private static final List<EncodingProfile> ENCODING_PROFILES = Arrays.asList(
        new EncodingProfile("360p", 640, 360, 1_000_000, "libx264"),
        new EncodingProfile("480p", 854, 480, 2_000_000, "libx264"),
        new EncodingProfile("720p", 1280, 720, 5_000_000, "libx264"),
        new EncodingProfile("1080p", 1920, 1080, 8_000_000, "libx264"),
        new EncodingProfile("4K", 3840, 2160, 25_000_000, "libx265")
    );
    
    public UUID submitEncodingJob(String videoPath, UUID contentId) {
        UUID jobId = UUID.randomUUID();
        
        // Create encoding job
        EncodingJob job = EncodingJob.builder()
            .jobId(jobId)
            .contentId(contentId)
            .sourceVideoPath(videoPath)
            .status(EncodingStatus.PENDING)
            .profiles(ENCODING_PROFILES)
            .createdAt(Instant.now())
            .build();
        
        // Save to DynamoDB
        dynamoClient.putItem(job.toDynamoItem());
        
        // Queue for processing
        sqsClient.sendMessage(new EncodingJobMessage(jobId).toJson());
        
        return jobId;
    }
    
    public void processEncodingJob(UUID jobId) {
        // This runs on worker nodes
        EncodingJob job = getJob(jobId);
        
        try {
            updateJobStatus(jobId, EncodingStatus.PROCESSING);
            
            // 1. Download source video from S3
            File sourceFile = downloadFromS3(job.getSourceVideoPath());
            
            // 2. Encode to multiple qualities
            List<VideoFile> encodedFiles = new ArrayList<>();
            int totalProfiles = job.getProfiles().size();
            
            for (int i = 0; i < totalProfiles; i++) {
                EncodingProfile profile = job.getProfiles().get(i);
                
                // Update progress
                updateJobProgress(jobId, (i * 100) / totalProfiles);
                
                // Encode
                File encodedFile = encodeVideo(sourceFile, profile);
                
                // Generate HLS segments
                List<File> hlsSegments = generateHLSSegments(
                    encodedFile,
                    profile
                );
                
                // Upload to S3
                String s3Path = uploadToS3(
                    hlsSegments,
                    job.getContentId(),
                    profile.getQuality()
                );
                
                // Create video file record
                VideoFile videoFile = VideoFile.builder()
                    .quality(profile.getQuality())
                    .resolution(profile.getWidth() + "x" + profile.getHeight())
                    .bitrate(profile.getBitrate())
                    .codec(profile.getCodec())
                    .storageLocation(s3Path)
                    .build();
                
                encodedFiles.add(videoFile);
                
                // Cleanup temp files
                encodedFile.delete();
            }
            
            // 3. Generate master HLS manifest
            String manifestContent = generateMasterManifest(encodedFiles);
            String manifestPath = uploadManifestToS3(
                manifestContent,
                job.getContentId()
            );
            
            // 4. Generate thumbnails
            List<String> thumbnails = generateThumbnails(
                sourceFile,
                job.getContentId()
            );
            
            // 5. Generate preview clip (first 30 seconds)
            File previewClip = generatePreview(sourceFile);
            String previewPath = uploadToS3(
                previewClip,
                job.getContentId(),
                "preview"
            );
            
            // 6. Save video metadata to database
            saveVideoMetadata(
                job.getContentId(),
                encodedFiles,
                manifestPath,
                thumbnails,
                previewPath
            );
            
            // 7. Invalidate CDN cache (push to edge servers)
            cdnManager.invalidate(job.getContentId());
            
            // 8. Mark job as completed
            updateJobStatus(jobId, EncodingStatus.COMPLETED);
            
            // Cleanup
            sourceFile.delete();
            
        } catch (Exception e) {
            updateJobStatus(jobId, EncodingStatus.FAILED);
            throw new EncodingException("Encoding failed for job: " + jobId, e);
        }
    }
    
    private File encodeVideo(File sourceFile, EncodingProfile profile) {
        // Use FFmpeg for encoding
        String outputPath = "/tmp/" + UUID.randomUUID() + ".mp4";
        
        String[] command = {
            "ffmpeg",
            "-i", sourceFile.getAbsolutePath(),
            "-vf", String.format("scale=%d:%d", profile.getWidth(), profile.getHeight()),
            "-c:v", profile.getCodec(),
            "-b:v", String.valueOf(profile.getBitrate()),
            "-c:a", "aac",
            "-b:a", "128k",
            "-movflags", "+faststart",
            "-preset", "slow", // Better compression
            "-crf", "23", // Quality factor
            outputPath
        };
        
        try {
            Process process = new ProcessBuilder(command).start();
            int exitCode = process.waitFor();
            
            if (exitCode != 0) {
                throw new RuntimeException("FFmpeg encoding failed");
            }
            
            return new File(outputPath);
        } catch (Exception e) {
            throw new EncodingException("Video encoding failed", e);
        }
    }
    
    private List<File> generateHLSSegments(File videoFile, EncodingProfile profile) {
        String outputDir = "/tmp/" + UUID.randomUUID() + "/";
        new File(outputDir).mkdirs();
        
        String[] command = {
            "ffmpeg",
            "-i", videoFile.getAbsolutePath(),
            "-c", "copy",
            "-hls_time", "10", // 10 second segments
            "-hls_list_size", "0",
            "-hls_segment_filename", outputDir + "segment_%03d.ts",
            outputDir + "playlist.m3u8"
        };
        
        try {
            Process process = new ProcessBuilder(command).start();
            process.waitFor();
            
            // Return all generated files
            return Arrays.asList(new File(outputDir).listFiles());
        } catch (Exception e) {
            throw new EncodingException("HLS generation failed", e);
        }
    }
    
    private String generateMasterManifest(List<VideoFile> videoFiles) {
        StringBuilder manifest = new StringBuilder();
        manifest.append("#EXTM3U\n");
        manifest.append("#EXT-X-VERSION:3\n\n");
        
        for (VideoFile vf : videoFiles) {
            manifest.append(String.format(
                "#EXT-X-STREAM-INF:BANDWIDTH=%d,RESOLUTION=%s\n",
                vf.getBitrate(),
                vf.getResolution()
            ));
            manifest.append(vf.getStorageLocation() + "/playlist.m3u8\n");
        }
        
        return manifest.toString();
    }
    
    private List<String> generateThumbnails(File videoFile, UUID contentId) {
        // Generate thumbnails at different timestamps
        List<String> thumbnailUrls = new ArrayList<>();
        int[] timestamps = {0, 30, 60, 120, 180}; // seconds
        
        for (int ts : timestamps) {
            String outputPath = "/tmp/thumb_" + ts + ".jpg";
            
            String[] command = {
                "ffmpeg",
                "-ss", String.valueOf(ts),
                "-i", videoFile.getAbsolutePath(),
                "-vframes", "1",
                "-q:v", "2",
                "-vf", "scale=1280:720",
                outputPath
            };
            
            try {
                Process process = new ProcessBuilder(command).start();
                process.waitFor();
                
                // Upload to S3
                String s3Url = uploadToS3(
                    new File(outputPath),
                    contentId,
                    "thumbnails/thumb_" + ts + ".jpg"
                );
                thumbnailUrls.add(s3Url);
                
                new File(outputPath).delete();
            } catch (Exception e) {
                // Log and continue
            }
        }
        
        return thumbnailUrls;
    }
}
```

#### 7.2.3 Recommendation Service

```java
public class RecommendationService {
    private final TensorFlowServingClient mlClient;
    private final WatchHistoryRepository watchHistoryRepo;
    private final ContentRepository contentRepo;
    private final CacheManager cacheManager;
    
    public List<RecommendedContent> getPersonalizedRecommendations(
        UUID profileId,
        int limit
    ) {
        // Try cache first
        String cacheKey = "recommendations:" + profileId;
        List<RecommendedContent> cached = cacheManager.get(cacheKey);
        if (cached != null) {
            return cached.subList(0, Math.min(limit, cached.size()));
        }
        
        // 1. Get user's watch history
        List<WatchHistory> history = watchHistoryRepo.findByProfileId(
            profileId,
            100 // last 100 items
        );
        
        if (history.isEmpty()) {
            // New user, return trending content
            return getTrendingContent(limit);
        }
        
        // 2. Extract features
        UserFeatures userFeatures = extractUserFeatures(history);
        
        // 3. Get candidate content (not already watched)
        Set<UUID> watchedContentIds = history.stream()
            .map(WatchHistory::getContentId)
            .collect(Collectors.toSet());
        
        List<Content> candidates = contentRepo.findAll().stream()
            .filter(c -> !watchedContentIds.contains(c.getContentId()))
            .collect(Collectors.toList());
        
        // 4. Score each candidate using ML model
        List<ScoredContent> scoredCandidates = new ArrayList<>();
        
        for (Content content : candidates) {
            ContentFeatures contentFeatures = extractContentFeatures(content);
            
            // Call ML model for prediction
            float score = mlClient.predict(userFeatures, contentFeatures);
            
            scoredCandidates.add(new ScoredContent(content, score));
        }
        
        // 5. Sort by score and take top N
        List<RecommendedContent> recommendations = scoredCandidates.stream()
            .sorted(Comparator.comparingDouble(ScoredContent::getScore).reversed())
            .limit(limit * 2) // Get more for diversity
            .map(sc -> new RecommendedContent(
                sc.getContent(),
                sc.getScore(),
                generateReason(sc.getContent(), history)
            ))
            .collect(Collectors.toList());
        
        // 6. Apply diversity (not all action movies)
        recommendations = applyDiversity(recommendations, limit);
        
        // 7. Cache results
        cacheManager.set(cacheKey, recommendations, Duration.ofMinutes(10));
        
        return recommendations;
    }
    
    private UserFeatures extractUserFeatures(List<WatchHistory> history) {
        // Extract aggregate features
        Map<String, Integer> genreCounts = new HashMap<>();
        Map<String, Integer> actorCounts = new HashMap<>();
        double avgRating = 0;
        double avgCompletionRate = 0;
        
        for (WatchHistory wh : history) {
            Content content = contentRepo.findById(wh.getContentId()).orElse(null);
            if (content == null) continue;
            
            // Genre preferences
            for (String genre : content.getGenres()) {
                genreCounts.merge(genre, 1, Integer::sum);
            }
            
            // Actor preferences
            for (String actor : content.getCast()) {
                actorCounts.merge(actor, 1, Integer::sum);
            }
            
            // Completion rate
            avgCompletionRate += (double) wh.getWatchedDuration() / wh.getTotalDuration();
        }
        
        avgCompletionRate /= history.size();
        
        return UserFeatures.builder()
            .genrePreferences(genreCounts)
            .actorPreferences(actorCounts)
            .avgCompletionRate(avgCompletionRate)
            .totalWatched(history.size())
            .build();
    }
    
    private ContentFeatures extractContentFeatures(Content content) {
        return ContentFeatures.builder()
            .genres(content.getGenres())
            .rating(content.getRating())
            .releaseYear(content.getReleaseYear())
            .cast(content.getCast())
            .director(content.getDirector())
            .popularity(content.getViewCount())
            .averageRating(content.getAverageRating())
            .build();
    }
    
    private List<RecommendedContent> applyDiversity(
        List<RecommendedContent> recommendations,
        int limit
    ) {
        // Ensure genre diversity
        List<RecommendedContent> diverse = new ArrayList<>();
        Set<String> usedGenres = new HashSet<>();
        
        // First pass: one from each genre
        for (RecommendedContent rec : recommendations) {
            String primaryGenre = rec.getContent().getGenres().get(0);
            if (!usedGenres.contains(primaryGenre)) {
                diverse.add(rec);
                usedGenres.add(primaryGenre);
                if (diverse.size() >= limit) break;
            }
        }
        
        // Second pass: fill remaining slots with highest scores
        if (diverse.size() < limit) {
            recommendations.stream()
                .filter(rec -> !diverse.contains(rec))
                .limit(limit - diverse.size())
                .forEach(diverse::add);
        }
        
        return diverse;
    }
    
    private String generateReason(Content content, List<WatchHistory> history) {
        // Generate human-readable reason
        // "Because you watched X"
        // "Trending in your area"
        // "Top pick for you"
        
        // Find most similar watched content
        for (WatchHistory wh : history) {
            Content watched = contentRepo.findById(wh.getContentId()).orElse(null);
            if (watched == null) continue;
            
            // Check for common genres
            Set<String> commonGenres = new HashSet<>(watched.getGenres());
            commonGenres.retainAll(content.getGenres());
            
            if (!commonGenres.isEmpty()) {
                return "Because you watched " + watched.getTitle();
            }
            
            // Check for common cast
            Set<String> commonCast = new HashSet<>(watched.getCast());
            commonCast.retainAll(content.getCast());
            
            if (!commonCast.isEmpty()) {
                return "More featuring " + commonCast.iterator().next();
            }
        }
        
        return "Top pick for you";
    }
    
    public List<RecommendedContent> getTrendingContent(int limit) {
        // Get content with highest view count in last 7 days
        return contentRepo.findTrending(Duration.ofDays(7), limit);
    }
}
```

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

