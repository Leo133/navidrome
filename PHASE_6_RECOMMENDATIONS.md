# Phase 6: Recommendations & Discovery
**Duration:** 4-5 weeks  
**Team Size:** 5-6 engineers (including ML engineer)  
**Dependencies:** Phase 5 complete (analytics operational)

---

## Objective

Build **intelligent recommendation engine** for music discovery:
1. Collaborative filtering (find users with similar taste)
2. Content-based recommendations (similar tracks/artists/genres)
3. Friend-based discovery (what are friends listening to)
4. "Discover Weekly" style playlists (personalized weekly updates)
5. Mood-based recommendations ("energetic", "chill", "focus")
6. Similar tracks/artists (based on listening patterns + metadata)

This creates a **Spotify/Last.fm-style recommendation system** for self-hosted music.

---

## Scope

### In Scope
- ✅ Recommendation Service (external microservice)
- ✅ Taste vectors (user preference embeddings)
- ✅ Collaborative filtering (user-to-user similarity)
- ✅ Content-based filtering (track/artist metadata)
- ✅ Friend recommendations (social graph + listening history)
- ✅ Discover Weekly playlist generation
- ✅ Similar tracks/artists API
- ✅ Mood-based recommendations

### Out of Scope
- ❌ Deep learning models (defer to later phase)
- ❌ Audio fingerprinting (use metadata only for v1)
- ❌ External data sources (Spotify API, Last.fm) - defer
- ❌ Real-time recommendations (batch processing for v1)

---

## Architecture

### Technology Stack

- **Language:** Python 3.11+ (better ML ecosystem than Go)
- **Framework:** FastAPI (async HTTP server)
- **ML Libraries:**
  - **scikit-learn:** Collaborative filtering, similarity metrics
  - **pandas/numpy:** Data processing
  - **scipy:** Sparse matrix operations
- **Vector Database:** Qdrant (efficient similarity search)
- **Database:** PostgreSQL (user preferences, recommendation cache)
- **Job Queue:** Celery + Redis (background tasks)

**Why Python:**
- Better ML ecosystem (scikit-learn, TensorFlow, PyTorch)
- Easier to prototype recommendation algorithms
- Pandas for data manipulation
- Can still integrate with Go services via HTTP/NATS

### Recommendation Approaches

**1. Collaborative Filtering (User-Based)**

```
Find users with similar listening history:
  - User A listened to: [Track1, Track2, Track3, Track4]
  - User B listened to: [Track1, Track2, Track3, Track5]
  - Similarity: 75% (3 common tracks / 4 unique tracks)
  - Recommend Track5 to User A
```

**2. Collaborative Filtering (Item-Based)**

```
Find tracks that are often listened to together:
  - Track1 is often followed by Track2 (60% of users)
  - Track1 is often followed by Track3 (40% of users)
  - If user listens to Track1, recommend Track2
```

**3. Content-Based Filtering**

```
Find tracks with similar metadata:
  - Track A: genre=rock, artist=Queen, year=1975, tempo=fast
  - Track B: genre=rock, artist=Led Zeppelin, year=1971, tempo=fast
  - Similarity: 80% (same genre, similar artist style, similar tempo)
  - If user likes Track A, recommend Track B
```

**4. Hybrid Approach (Combine all methods)**

```
Final score = 
  0.4 * collaborative_score +
  0.3 * content_score +
  0.2 * friend_score +
  0.1 * popularity_score
```

---

## Key Technical Work Items

### 1. Recommendation Service Setup

**Goal:** Bootstrap Python service with FastAPI, Celery, Qdrant.

**Project Structure:**

```
recommendation-service/
├── app/
│   ├── main.py                     # FastAPI app
│   ├── models/
│   │   ├── user_taste.py
│   │   ├── recommendation.py
│   │   └── track_embedding.py
│   ├── services/
│   │   ├── collaborative_filtering.py
│   │   ├── content_filtering.py
│   │   ├── friend_recommendations.py
│   │   └── hybrid_recommender.py
│   ├── tasks/
│   │   ├── celery_app.py
│   │   ├── build_taste_vectors.py
│   │   ├── compute_similarities.py
│   │   └── generate_discover_weekly.py
│   ├── api/
│   │   ├── routes.py
│   │   └── dependencies.py
│   └── db/
│       ├── postgres.py
│       └── qdrant.py
├── tests/
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
└── README.md
```

**Dependencies:** `requirements.txt`

```txt
fastapi==0.109.0
uvicorn[standard]==0.27.0
celery==5.3.4
redis==5.0.1
psycopg2-binary==2.9.9
sqlalchemy==2.0.25
qdrant-client==1.7.3
scikit-learn==1.4.0
pandas==2.1.4
numpy==1.26.3
scipy==1.11.4
python-dotenv==1.0.0
pydantic==2.5.3
pydantic-settings==2.1.0
```

**Configuration:** `.env`

```env
DATABASE_URL=postgresql://user:pass@localhost:5432/recommendations
REDIS_URL=redis://localhost:6379/0
QDRANT_URL=http://localhost:6333
NAVIDROME_API_URL=http://localhost:4533
SOCIAL_API_URL=http://localhost:8080/graphql
ANALYTICS_API_URL=http://localhost:8081/graphql
```

**Main Entry Point:** `app/main.py`

```python
from fastapi import FastAPI
from app.api import routes
from app.db import postgres, qdrant

app = FastAPI(title="Navidrome Recommendation Service")

@app.on_event("startup")
async def startup():
    await postgres.connect()
    await qdrant.connect()

@app.on_event("shutdown")
async def shutdown():
    await postgres.disconnect()
    await qdrant.disconnect()

app.include_router(routes.router, prefix="/api")

@app.get("/health")
async def health():
    return {"status": "healthy"}
```

**Celery Worker:** `app/tasks/celery_app.py`

```python
from celery import Celery
from celery.schedules import crontab

celery_app = Celery(
    "recommendation_tasks",
    broker="redis://localhost:6379/0",
    backend="redis://localhost:6379/0"
)

celery_app.conf.beat_schedule = {
    "build-taste-vectors": {
        "task": "app.tasks.build_taste_vectors.build_all_taste_vectors",
        "schedule": crontab(hour=2, minute=0),  # Daily at 2 AM
    },
    "generate-discover-weekly": {
        "task": "app.tasks.generate_discover_weekly.generate_all_playlists",
        "schedule": crontab(day_of_week=1, hour=3, minute=0),  # Monday at 3 AM
    },
}
```

**Tasks:**
- [ ] Create project structure
- [ ] Setup FastAPI app
- [ ] Setup Celery worker
- [ ] Setup Docker Compose (PostgreSQL, Redis, Qdrant)
- [ ] Test service startup

**Deliverable:** Runnable Recommendation Service

---

### 2. Taste Vectors (User Embeddings)

**Goal:** Build numerical representation of user's musical taste.

**Approach:**

```python
# User taste vector = weighted combination of listened tracks

Taste Vector (100 dimensions):
  - Dimensions 0-19: Genre preferences (rock=0.8, jazz=0.2, ...)
  - Dimensions 20-39: Artist preferences (Queen=0.9, Beatles=0.7, ...)
  - Dimensions 40-59: Tempo preferences (fast=0.6, slow=0.4, ...)
  - Dimensions 60-79: Year preferences (70s=0.7, 80s=0.5, ...)
  - Dimensions 80-99: Mood preferences (energetic=0.8, chill=0.3, ...)

Example:
  User A: [0.8, 0.2, 0.1, ..., 0.6, 0.4]  (loves rock, prefers fast tempo)
  User B: [0.2, 0.8, 0.7, ..., 0.4, 0.6]  (loves jazz, prefers slow tempo)
  Similarity: cosine_similarity(A, B) = 0.35 (not very similar)
```

**Implementation:** `app/services/taste_vector_builder.py`

```python
import numpy as np
import pandas as pd
from sklearn.preprocessing import normalize
from typing import Dict, List

class TasteVectorBuilder:
    def __init__(self, analytics_client, navidrome_client):
        self.analytics_client = analytics_client
        self.navidrome_client = navidrome_client
        self.vector_size = 100
        
    async def build_taste_vector(self, user_id: str) -> np.ndarray:
        """Build taste vector for a user based on listening history."""
        
        # Get user's listening history (last 90 days)
        intervals = await self.analytics_client.get_user_intervals(
            user_id, 
            since=datetime.now() - timedelta(days=90)
        )
        
        if not intervals:
            return np.zeros(self.vector_size)  # No data, return zero vector
        
        # Aggregate listen time by track
        track_listen_times = {}
        for interval in intervals:
            track_id = interval["mediaFileId"]
            duration = interval["durationMs"]
            track_listen_times[track_id] = track_listen_times.get(track_id, 0) + duration
        
        # Get track metadata (genre, artist, tempo, year, etc.)
        tracks_metadata = await self.navidrome_client.get_tracks_bulk(
            list(track_listen_times.keys())
        )
        
        # Build vector
        vector = np.zeros(self.vector_size)
        
        total_listen_time = sum(track_listen_times.values())
        
        for track_id, listen_time in track_listen_times.items():
            weight = listen_time / total_listen_time
            metadata = tracks_metadata.get(track_id, {})
            
            # Genre (dimensions 0-19)
            genre = metadata.get("genre", "unknown")
            genre_idx = self._genre_to_index(genre)
            vector[genre_idx] += weight
            
            # Artist (dimensions 20-39) - use artist ID hash
            artist = metadata.get("artist", "")
            artist_idx = 20 + (hash(artist) % 20)
            vector[artist_idx] += weight
            
            # Tempo (dimensions 40-59)
            tempo = metadata.get("bpm", 120)
            tempo_idx = 40 + self._tempo_to_index(tempo)
            vector[tempo_idx] += weight
            
            # Year (dimensions 60-79)
            year = metadata.get("year", 2000)
            year_idx = 60 + self._year_to_index(year)
            vector[year_idx] += weight
            
            # Mood (dimensions 80-99) - derived from genre/tempo
            mood_idx = 80 + self._infer_mood_index(genre, tempo)
            vector[mood_idx] += weight
        
        # Normalize to unit vector
        vector = normalize(vector.reshape(1, -1))[0]
        
        return vector
    
    def _genre_to_index(self, genre: str) -> int:
        """Map genre to index (0-19)."""
        genre_map = {
            "rock": 0, "pop": 1, "jazz": 2, "classical": 3,
            "electronic": 4, "hip-hop": 5, "metal": 6, "folk": 7,
            "blues": 8, "country": 9, "r&b": 10, "reggae": 11,
            "punk": 12, "indie": 13, "alternative": 14, "soul": 15,
            "funk": 16, "disco": 17, "ambient": 18, "unknown": 19
        }
        return genre_map.get(genre.lower(), 19)
    
    def _tempo_to_index(self, bpm: int) -> int:
        """Map BPM to index (0-19)."""
        # Slow (0-79), medium (80-119), fast (120-159), very fast (160+)
        if bpm < 80:
            return 0
        elif bpm < 120:
            return 5
        elif bpm < 160:
            return 10
        else:
            return 15
    
    def _year_to_index(self, year: int) -> int:
        """Map year to index (0-19)."""
        # Decades: 50s, 60s, 70s, 80s, 90s, 00s, 10s, 20s
        decade = (year // 10) * 10
        decade_map = {1950: 0, 1960: 2, 1970: 4, 1980: 6, 1990: 8, 2000: 10, 2010: 12, 2020: 14}
        return decade_map.get(decade, 19)
    
    def _infer_mood_index(self, genre: str, tempo: int) -> int:
        """Infer mood from genre and tempo (0-19)."""
        # Energetic (0-4), Happy (5-9), Chill (10-14), Sad (15-19)
        if genre in ["rock", "metal", "punk", "electronic"] or tempo > 140:
            return 2  # Energetic
        elif genre in ["pop", "disco", "funk"] or tempo > 120:
            return 7  # Happy
        elif genre in ["ambient", "jazz", "classical"] or tempo < 80:
            return 12  # Chill
        else:
            return 17  # Neutral/Sad
```

**Celery Task:** `app/tasks/build_taste_vectors.py`

```python
from app.tasks.celery_app import celery_app
from app.services.taste_vector_builder import TasteVectorBuilder
from app.db.qdrant import qdrant_client
from app.db.postgres import get_all_user_ids

@celery_app.task
def build_all_taste_vectors():
    """Build taste vectors for all users (runs daily)."""
    
    builder = TasteVectorBuilder(analytics_client, navidrome_client)
    user_ids = get_all_user_ids()
    
    for user_id in user_ids:
        try:
            # Build taste vector
            vector = await builder.build_taste_vector(user_id)
            
            # Store in Qdrant (vector database)
            await qdrant_client.upsert(
                collection_name="user_taste_vectors",
                points=[{
                    "id": user_id,
                    "vector": vector.tolist(),
                    "payload": {"user_id": user_id}
                }]
            )
            
            print(f"Built taste vector for user {user_id}")
        except Exception as e:
            print(f"Failed to build taste vector for user {user_id}: {e}")
    
    print(f"Built taste vectors for {len(user_ids)} users")
```

**Tasks:**
- [ ] Implement taste vector builder
- [ ] Create Celery task (build_all_taste_vectors)
- [ ] Setup Qdrant collection (user_taste_vectors)
- [ ] Test with real user data

**Deliverable:** Taste vector generation

**Acceptance Criteria:**
- Taste vectors generated for all users (daily)
- Vectors are normalized (unit length)
- Vectors stored in Qdrant (efficient similarity search)

---

### 3. Collaborative Filtering

**Goal:** Find users with similar taste and recommend their favorite tracks.

**Implementation:** `app/services/collaborative_filtering.py`

```python
from qdrant_client import QdrantClient
from typing import List, Tuple

class CollaborativeFilter:
    def __init__(self, qdrant_client: QdrantClient, analytics_client):
        self.qdrant_client = qdrant_client
        self.analytics_client = analytics_client
        
    async def get_similar_users(self, user_id: str, limit: int = 50) -> List[Tuple[str, float]]:
        """Find users with similar taste (cosine similarity)."""
        
        # Get user's taste vector from Qdrant
        user_vector = await self.qdrant_client.retrieve(
            collection_name="user_taste_vectors",
            ids=[user_id]
        )
        
        if not user_vector:
            return []
        
        # Search for similar vectors
        similar = await self.qdrant_client.search(
            collection_name="user_taste_vectors",
            query_vector=user_vector[0].vector,
            limit=limit + 1,  # +1 to exclude self
            score_threshold=0.5  # Minimum similarity
        )
        
        # Filter out self
        similar_users = [
            (point.payload["user_id"], point.score)
            for point in similar
            if point.payload["user_id"] != user_id
        ]
        
        return similar_users[:limit]
    
    async def recommend_tracks(self, user_id: str, limit: int = 50) -> List[str]:
        """Recommend tracks based on similar users' listening history."""
        
        # Get similar users
        similar_users = await self.get_similar_users(user_id, limit=50)
        
        if not similar_users:
            return []
        
        # Get user's already-listened tracks (don't recommend duplicates)
        user_listened = await self.analytics_client.get_user_listened_tracks(user_id)
        user_listened_set = set(user_listened)
        
        # Aggregate recommendations from similar users (weighted by similarity)
        track_scores = {}
        
        for similar_user_id, similarity in similar_users:
            # Get similar user's favorite tracks (most listened)
            similar_user_tracks = await self.analytics_client.get_user_top_tracks(
                similar_user_id, 
                limit=100
            )
            
            for track_id, listen_count in similar_user_tracks.items():
                if track_id not in user_listened_set:
                    # Score = similarity * listen_count
                    score = similarity * listen_count
                    track_scores[track_id] = track_scores.get(track_id, 0) + score
        
        # Sort by score and return top N
        recommended = sorted(track_scores.items(), key=lambda x: x[1], reverse=True)
        return [track_id for track_id, score in recommended[:limit]]
```

**Tasks:**
- [ ] Implement collaborative filtering
- [ ] Test similarity search (verify similar users have similar listening history)
- [ ] Test recommendations (verify recommended tracks make sense)

**Deliverable:** Collaborative filtering recommendations

**Acceptance Criteria:**
- Similar users have >50% overlap in listening history
- Recommendations are tracks user hasn't listened to
- Recommendations are diverse (not all from same artist/album)

---

### 4. Friend-Based Recommendations

**Goal:** Recommend tracks that friends are listening to.

**Implementation:** `app/services/friend_recommendations.py`

```python
class FriendRecommender:
    def __init__(self, social_client, analytics_client):
        self.social_client = social_client
        self.analytics_client = analytics_client
        
    async def recommend_from_friends(self, user_id: str, limit: int = 50) -> List[str]:
        """Recommend tracks that friends are listening to."""
        
        # Get user's friends (people they follow)
        friends = await self.social_client.get_following(user_id)
        
        if not friends:
            return []
        
        # Get user's already-listened tracks
        user_listened = await self.analytics_client.get_user_listened_tracks(user_id)
        user_listened_set = set(user_listened)
        
        # Get friends' recent listens (last 7 days)
        track_scores = {}
        
        for friend_id in friends:
            friend_recent = await self.analytics_client.get_user_recent_tracks(
                friend_id, 
                days=7, 
                limit=100
            )
            
            for track_id, listen_count in friend_recent.items():
                if track_id not in user_listened_set:
                    # Score = listen_count (tracks friends listen to a lot)
                    track_scores[track_id] = track_scores.get(track_id, 0) + listen_count
        
        # Sort by score and return top N
        recommended = sorted(track_scores.items(), key=lambda x: x[1], reverse=True)
        return [track_id for track_id, score in recommended[:limit]]
```

**Tasks:**
- [ ] Implement friend recommendations
- [ ] Test with real friend data
- [ ] Verify recommendations are fresh (recent listens)

**Deliverable:** Friend-based recommendations

**Acceptance Criteria:**
- Recommendations are tracks friends listened to recently (<7 days)
- Recommendations are tracks user hasn't listened to
- Recommendations favor tracks multiple friends listened to

---

### 5. Hybrid Recommender (Combine All Methods)

**Goal:** Combine collaborative, content-based, and friend-based recommendations.

**Implementation:** `app/services/hybrid_recommender.py`

```python
class HybridRecommender:
    def __init__(self, collab_filter, content_filter, friend_recommender):
        self.collab_filter = collab_filter
        self.content_filter = content_filter
        self.friend_recommender = friend_recommender
        
    async def recommend(self, user_id: str, limit: int = 50) -> List[str]:
        """Hybrid recommendation (weighted combination)."""
        
        # Get recommendations from each method
        collab_recs = await self.collab_filter.recommend_tracks(user_id, limit=100)
        content_recs = await self.content_filter.recommend_tracks(user_id, limit=100)
        friend_recs = await self.friend_recommender.recommend_from_friends(user_id, limit=100)
        
        # Assign weights
        weights = {
            "collaborative": 0.4,
            "content": 0.3,
            "friend": 0.3
        }
        
        # Combine scores
        track_scores = {}
        
        for i, track_id in enumerate(collab_recs):
            score = weights["collaborative"] * (100 - i)  # Higher rank = higher score
            track_scores[track_id] = track_scores.get(track_id, 0) + score
        
        for i, track_id in enumerate(content_recs):
            score = weights["content"] * (100 - i)
            track_scores[track_id] = track_scores.get(track_id, 0) + score
        
        for i, track_id in enumerate(friend_recs):
            score = weights["friend"] * (100 - i)
            track_scores[track_id] = track_scores.get(track_id, 0) + score
        
        # Sort by final score
        recommended = sorted(track_scores.items(), key=lambda x: x[1], reverse=True)
        return [track_id for track_id, score in recommended[:limit]]
```

**Tasks:**
- [ ] Implement hybrid recommender
- [ ] Tune weights (A/B test different combinations)
- [ ] Validate recommendations (user feedback)

**Deliverable:** Hybrid recommendation system

**Acceptance Criteria:**
- Recommendations combine all methods
- Recommendations are diverse (from different methods)
- Recommendations are high-quality (>70% user satisfaction)

---

### 6. Discover Weekly Playlist Generation

**Goal:** Auto-generate personalized weekly playlists (Spotify-style).

**Implementation:** `app/tasks/generate_discover_weekly.py`

```python
@celery_app.task
def generate_all_playlists():
    """Generate Discover Weekly playlists for all users (runs every Monday)."""
    
    recommender = HybridRecommender(collab_filter, content_filter, friend_recommender)
    user_ids = get_all_user_ids()
    
    for user_id in user_ids:
        try:
            # Get recommendations (30 tracks)
            tracks = await recommender.recommend(user_id, limit=30)
            
            # Create playlist in Navidrome
            playlist = await navidrome_client.create_playlist(
                user_id=user_id,
                name=f"Discover Weekly - {datetime.now().strftime('%Y-%m-%d')}",
                tracks=tracks
            )
            
            print(f"Generated Discover Weekly for user {user_id}: {playlist['id']}")
        except Exception as e:
            print(f"Failed to generate playlist for user {user_id}: {e}")
    
    print(f"Generated Discover Weekly for {len(user_ids)} users")
```

**Tasks:**
- [ ] Implement playlist generation task
- [ ] Schedule Celery task (every Monday at 3 AM)
- [ ] Test with real users
- [ ] Add notification (new playlist available)

**Deliverable:** Discover Weekly playlist generation

**Acceptance Criteria:**
- Playlists generated every Monday
- Playlists contain 30 tracks (diverse, high-quality)
- Users are notified of new playlists

---

### 7. API Endpoints

**Goal:** Expose recommendations via REST API.

**Routes:** `app/api/routes.py`

```python
from fastapi import APIRouter, Depends
from app.services.hybrid_recommender import HybridRecommender

router = APIRouter()

@router.get("/recommendations/tracks")
async def get_track_recommendations(
    user_id: str,
    limit: int = 50,
    recommender: HybridRecommender = Depends()
):
    """Get personalized track recommendations."""
    tracks = await recommender.recommend(user_id, limit)
    return {"tracks": tracks}

@router.get("/recommendations/similar-users")
async def get_similar_users(
    user_id: str,
    limit: int = 50,
    collab_filter: CollaborativeFilter = Depends()
):
    """Get users with similar taste."""
    similar = await collab_filter.get_similar_users(user_id, limit)
    return {"similar_users": similar}

@router.get("/recommendations/similar-tracks/{track_id}")
async def get_similar_tracks(
    track_id: str,
    limit: int = 20,
    content_filter: ContentFilter = Depends()
):
    """Get tracks similar to a given track."""
    similar = await content_filter.get_similar_tracks(track_id, limit)
    return {"similar_tracks": similar}

@router.get("/recommendations/discover-weekly/{user_id}")
async def get_discover_weekly(
    user_id: str,
    navidrome_client = Depends()
):
    """Get user's Discover Weekly playlist."""
    playlists = await navidrome_client.get_playlists(user_id)
    discover_weekly = [p for p in playlists if "Discover Weekly" in p["name"]]
    return {"playlist": discover_weekly[0] if discover_weekly else None}
```

**Tasks:**
- [ ] Create API routes
- [ ] Add authentication (JWT verification)
- [ ] Add rate limiting
- [ ] Test endpoints

**Deliverable:** Recommendation API

**Acceptance Criteria:**
- Endpoints return valid recommendations
- Endpoints are authenticated (require JWT)
- Endpoints are performant (<500ms)

---

### 8. UI Integration

**Goal:** Add recommendation features to Navidrome UI.

**Components:**

**Recommended For You:** `ui/src/recommendations/RecommendedTracks.jsx`

```tsx
export const RecommendedTracks = () => {
    const [tracks, setTracks] = useState([]);
    
    useEffect(() => {
        fetchRecommendations();
    }, []);
    
    const fetchRecommendations = async () => {
        const response = await fetch('/api/recommendations/tracks?limit=20', {
            headers: { 'Authorization': `Bearer ${getToken()}` }
        });
        const data = await response.json();
        
        // Hydrate track details from Navidrome
        const trackDetails = await Promise.all(
            data.tracks.map(trackId => navidromeClient.getTrack(trackId))
        );
        
        setTracks(trackDetails);
    };
    
    return (
        <Card>
            <CardContent>
                <Typography variant="h5">Recommended For You</Typography>
                <TrackList tracks={tracks} />
            </CardContent>
        </Card>
    );
};
```

**Similar Tracks:** `ui/src/song/SimilarTracks.jsx`

```tsx
export const SimilarTracks = ({ trackId }) => {
    const [similar, setSimilar] = useState([]);
    
    useEffect(() => {
        fetchSimilar();
    }, [trackId]);
    
    const fetchSimilar = async () => {
        const response = await fetch(`/api/recommendations/similar-tracks/${trackId}`, {
            headers: { 'Authorization': `Bearer ${getToken()}` }
        });
        const data = await response.json();
        
        const trackDetails = await Promise.all(
            data.similar_tracks.map(trackId => navidrome Client.getTrack(trackId))
        );
        
        setSimilar(trackDetails);
    };
    
    return (
        <div>
            <Typography variant="h6">Similar Tracks</Typography>
            <TrackList tracks={similar} />
        </div>
    );
};
```

**Discover Weekly:** Add to playlists page (special badge)

**Tasks:**
- [ ] Create Recommended For You component (home page)
- [ ] Create Similar Tracks component (track detail page)
- [ ] Highlight Discover Weekly playlist
- [ ] Add "Why recommended?" tooltip (show method: collaborative/content/friend)

**Deliverable:** Recommendation UI

**Acceptance Criteria:**
- Recommended For You shows on home page
- Similar Tracks shows on track detail page
- Discover Weekly playlist is highlighted
- UI loads in <2 seconds

---

## Operational Needs

### Infrastructure
- Python runtime (Docker container)
- Qdrant (vector database) - managed service or self-hosted
- Celery workers (background tasks)
- Redis (Celery broker)

### Monitoring
- Prometheus metrics:
  - `recommendations_generated_total` (counter)
  - `taste_vectors_updated_total` (counter)
  - `recommendation_latency_seconds` (histogram)
- Alerts for:
  - Celery task failures
  - Qdrant downtime
  - Recommendation quality drop (user feedback <50% positive)

### Data Retention
- Taste vectors: Updated daily (no retention policy)
- Recommendation cache: 7 days (refresh weekly)

---

## Risks & Mitigations

### Risk: Cold start problem (new users have no data)
**Likelihood:** High  
**Impact:** Medium  
**Mitigation:**
- Fallback to popular tracks (most listened globally)
- Prompt new users to like/rate tracks (bootstrap taste profile)
- Use genre preferences from signup

### Risk: Recommendations are poor quality (low diversity)
**Likelihood:** Medium  
**Impact:** High  
**Mitigation:**
- Add diversity penalty (penalize too many tracks from same artist)
- A/B test different weights (collaborative vs. content vs. friend)
- Collect user feedback ("Was this recommendation helpful?")

### Risk: Scalability (taste vector computation is slow)
**Likelihood:** Low  
**Impact:** Medium  
**Mitigation:**
- Batch process (compute once daily, not on-demand)
- Cache recommendations (7-day TTL)
- Optimize vector operations (use sparse matrices)

---

## Definition of Done

### Must Have
- ✅ Recommendation Service deployed
- ✅ Taste vectors generated for all users
- ✅ Collaborative filtering working
- ✅ Friend-based recommendations working
- ✅ Hybrid recommender combining all methods
- ✅ Discover Weekly playlist generation (weekly)
- ✅ API endpoints for recommendations
- ✅ UI integration (Recommended For You, Similar Tracks)

### Should Have
- ✅ Content-based filtering (similar tracks by metadata)
- ✅ Mood-based recommendations
- ✅ "Why recommended?" explanations

### Could Have
- Deep learning models (audio fingerprinting) - defer
- External data sources (Spotify, Last.fm) - defer
- Real-time recommendations (WebSocket updates) - defer

---

## Success Metrics

- **Recommendation Adoption:** >50% of users click on at least 1 recommendation per week
- **Discover Weekly Engagement:** >30% of users play Discover Weekly playlist
- **Recommendation Quality:** >60% user satisfaction (based on feedback)
- **API Performance:** P95 <500ms for recommendation requests

---

## Next Steps (Transition to Phase 7)

After Phase 6 is complete:
1. Monitor recommendation quality (2 weeks)
2. Collect user feedback (survey, thumbs up/down)
3. A/B test recommendation weights
4. Create Phase 7 branch: `feature/hardening`
5. Begin work on security, privacy, and production readiness

---

**Estimated Effort:**
- Senior ML Engineer (Python): 120 hours (taste vectors, collaborative filtering, hybrid recommender)
- Mid-Level Backend Engineer (Python): 80 hours (API, Celery tasks, Qdrant integration)
- Senior Frontend Engineer (React): 60 hours (recommendation UI)
- Mid-Level Frontend Engineer (React): 40 hours (UI polish, loading states)
- **Total: 300 hours (~7.5 person-weeks with 40-hour weeks)**

---

**End of Phase 6**
