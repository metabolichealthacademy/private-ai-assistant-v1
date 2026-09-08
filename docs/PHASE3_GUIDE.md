# Phase 3: Intelligence - Implementation Guide

## Overview

Phase 3 transforms the AI Assistant into an intelligent decision-making system by implementing AI-driven analysis, pattern recognition, and personalized insights across email, calendar, and tasks.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│         Phase 2: Data Integration Layer              │
│    (Email, Calendar, Tasks, AI Models, Notifications)│
└────────────────────────┬────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Inbox Triage │  │ Priority     │  │  Calendar    │
│  Service     │  │ Detection    │  │  Analysis    │
└──────────────┘  └──────────────┘  └──────────────┘
        │                │                │
        │      ┌─────────┼─────────┐      │
        │      │                  │      │
        ▼      ▼                  ▼      ▼
┌──────────────────────────────────────────────┐
│      Intelligence Orchestrator Service        │
│  (Coordinates all intelligence operations)    │
└──────────────┬─────────────────────────────┘
               │
        ┌──────┼──────┐
        │      │      │
        ▼      ▼      ▼
┌────────┐ ┌────────┐ ┌─────────────┐
│  Task  │ │ Daily  │ │  Machine    │
│Priorit.│ │Briefing│ │  Learning   │
│ Service│ │Service │ │  Models     │
└────────┘ └────────┘ └─────────────┘
        │      │      │
        └──────┼──────┘
               │
        ┌──────┴──────┐
        │             │
        ▼             ▼
    ┌────────┐   ┌─────────┐
    │Database│   │  Cache  │
    │(Phase2)│   │ (Redis) │
    └────────┘   └─────────┘
```

## Component 3.1: Inbox Triage Service

### Purpose
Automatically categorize incoming messages and score them for urgency and relevance.

### Database Schema

```sql
-- Message classification
CREATE TABLE message_classifications (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    message_id VARCHAR(255) NOT NULL,
    source VARCHAR(50) NOT NULL, -- 'email', 'slack', 'teams'
    category VARCHAR(100), -- 'urgent', 'important', 'followup', 'spam', 'archive'
    urgency_score DECIMAL(3,2), -- 0.0-1.0
    relevance_score DECIMAL(3,2), -- 0.0-1.0
    sender_importance DECIMAL(3,2), -- Based on historical patterns
    confidence DECIMAL(3,2), -- ML model confidence
    recommended_action VARCHAR(100), -- 'reply', 'archive', 'follow_up', 'delegate'
    tags JSONB, -- Auto-generated tags
    is_spam BOOLEAN DEFAULT false,
    manual_override BOOLEAN DEFAULT false,
    human_classification VARCHAR(100),
    classification_model VARCHAR(50), -- Model used for classification
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Classification feedback for model training
CREATE TABLE classification_feedback (
    id SERIAL PRIMARY KEY,
    classification_id INTEGER REFERENCES message_classifications(id),
    user_id INTEGER REFERENCES users(id),
    actual_category VARCHAR(100),
    feedback_type VARCHAR(50), -- 'correct', 'incorrect', 'unsure'
    improvement_notes TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Sender profiles for importance tracking
CREATE TABLE sender_profiles (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    sender_email VARCHAR(255),
    sender_name VARCHAR(255),
    interaction_count INTEGER DEFAULT 0,
    average_response_time INTERVAL,
    importance_score DECIMAL(3,2), -- 0.0-1.0
    category VARCHAR(100), -- 'close_contact', 'colleague', 'client', 'vendor'
    last_interaction TIMESTAMP,
    is_blocked BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

### Service Implementation

```python
# src/services/intelligence/inbox_triage.py
from typing import Dict, List, Optional
from datetime import datetime
from sqlalchemy.orm import Session
import numpy as np
from sklearn.pipeline import Pipeline
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.ensemble import RandomForestClassifier
import logging

logger = logging.getLogger(__name__)

class InboxTriageService:
    """Intelligent inbox categorization and prioritization"""
    
    def __init__(self, db: Session):
        self.db = db
        self.model = self._load_or_train_model()
    
    async def analyze_message(
        self,
        user_id: int,
        message_id: str,
        content: str,
        sender: str,
        subject: str,
        source: str = "email"
    ) -> Dict:
        """Analyze and classify incoming message"""
        
        try:
            # 1. Extract features
            features = await self._extract_features(
                user_id, content, sender, subject, source
            )
            
            # 2. Score urgency and relevance
            urgency_score = await self._score_urgency(
                user_id, features, content
            )
            relevance_score = await self._score_relevance(
                user_id, features, content
            )
            
            # 3. Detect spam
            is_spam = await self._detect_spam(content, sender)
            
            # 4. Get sender importance
            sender_importance = await self._get_sender_importance(
                user_id, sender
            )
            
            # 5. Predict category
            category = await self._predict_category(
                user_id, features, content, urgency_score, relevance_score
            )
            
            # 6. Recommend action
            recommended_action = self._recommend_action(
                category, urgency_score, relevance_score, is_spam
            )
            
            # 7. Generate tags
            tags = await self._generate_tags(content, subject)
            
            # 8. Store classification
            classification = await self._store_classification(
                user_id, message_id, {
                    "source": source,
                    "category": category,
                    "urgency_score": urgency_score,
                    "relevance_score": relevance_score,
                    "sender_importance": sender_importance,
                    "is_spam": is_spam,
                    "recommended_action": recommended_action,
                    "tags": tags
                }
            )
            
            return {
                "message_id": message_id,
                "category": category,
                "urgency_score": float(urgency_score),
                "relevance_score": float(relevance_score),
                "sender_importance": float(sender_importance),
                "is_spam": is_spam,
                "recommended_action": recommended_action,
                "tags": tags,
                "confidence": float(max(urgency_score, relevance_score))
            }
        
        except Exception as e:
            logger.error(f"Message analysis failed: {str(e)}")
            raise
    
    async def _extract_features(
        self,
        user_id: int,
        content: str,
        sender: str,
        subject: str,
        source: str
    ) -> Dict:
        """Extract ML features from message"""
        
        features = {
            "content_length": len(content),
            "subject_length": len(subject),
            "has_urgency_keywords": self._check_urgency_keywords(content),
            "has_action_items": self._check_action_items(content),
            "has_question": "?" in content,
            "all_caps_words": sum(1 for w in content.split() if w.isupper()),
            "exclamation_marks": content.count("!"),
            "contains_numbers": any(c.isdigit() for c in content),
            "sender_domain": sender.split("@")[1] if "@" in sender else "",
        }
        
        return features
    
    async def _score_urgency(
        self,
        user_id: int,
        features: Dict,
        content: str
    ) -> float:
        """Score message urgency (0.0-1.0)"""
        
        urgency_keywords = [
            "urgent", "asap", "immediately", "critical", "emergency",
            "deadline", "time-sensitive", "hurry", "rush"
        ]
        
        score = 0.0
        
        # Keyword scoring
        content_lower = content.lower()
        keyword_matches = sum(1 for kw in urgency_keywords if kw in content_lower)
        score += min(keyword_matches * 0.15, 0.6)
        
        # Feature scoring
        if features.get("all_caps_words", 0) > 5:
            score += 0.2
        if features.get("exclamation_marks", 0) > 2:
            score += 0.15
        if features.get("has_question"):
            score += 0.05
        
        return min(score, 1.0)
    
    async def _score_relevance(
        self,
        user_id: int,
        features: Dict,
        content: str
    ) -> float:
        """Score message relevance to user (0.0-1.0)"""
        
        # Get user's interests/keywords from preferences
        user_preferences = await self._get_user_preferences(user_id)
        relevant_keywords = user_preferences.get("keywords", [])
        
        score = 0.0
        
        # Keyword matching
        content_lower = content.lower()
        keyword_matches = sum(
            1 for kw in relevant_keywords 
            if kw.lower() in content_lower
        )
        score += min(keyword_matches * 0.2, 0.8)
        
        # Content quality scoring
        if features.get("content_length", 0) > 100:
            score += 0.1
        if features.get("has_action_items"):
            score += 0.1
        
        return min(score, 1.0)
    
    async def _detect_spam(self, content: str, sender: str) -> bool:
        """Detect if message is spam"""
        
        spam_indicators = [
            "click here", "limited time", "act now", "verify account",
            "confirm identity", "update payment", "suspicious activity"
        ]
        
        content_lower = content.lower()
        indicator_count = sum(1 for ind in spam_indicators if ind in content_lower)
        
        # Sender reputation check
        sender_is_blocked = await self._is_sender_blocked(sender)
        
        return indicator_count >= 2 or sender_is_blocked
    
    async def _get_sender_importance(
        self,
        user_id: int,
        sender: str
    ) -> float:
        """Calculate sender importance score"""
        
        profile = self.db.query(SenderProfile).filter(
            SenderProfile.user_id == user_id,
            SenderProfile.sender_email == sender
        ).first()
        
        if not profile:
            # Default score for new senders
            return 0.3
        
        # Calculate based on interaction history
        importance = (
            min(profile.interaction_count / 10, 0.4) +  # Interaction frequency
            (profile.importance_score or 0.0)  # Historical importance
        )
        
        return min(importance, 1.0)
    
    async def _predict_category(
        self,
        user_id: int,
        features: Dict,
        content: str,
        urgency_score: float,
        relevance_score: float
    ) -> str:
        """Predict message category"""
        
        # Use ensemble of heuristics and ML model
        if urgency_score > 0.7 and relevance_score > 0.5:
            return "urgent"
        elif relevance_score > 0.7:
            return "important"
        elif urgency_score > 0.5:
            return "followup"
        else:
            return "archive"
    
    def _recommend_action(
        self,
        category: str,
        urgency_score: float,
        relevance_score: float,
        is_spam: bool
    ) -> str:
        """Recommend action for message"""
        
        if is_spam:
            return "archive"
        elif category == "urgent":
            return "reply_now"
        elif category == "important":
            return "reply_soon"
        elif category == "followup":
            return "schedule_followup"
        else:
            return "archive"
    
    async def _generate_tags(self, content: str, subject: str) -> List[str]:
        """Generate tags from message content"""
        
        tags = []
        
        # Extract project names, client names, etc.
        # This would use NLP/keyword extraction
        keywords = await self._extract_keywords(content, subject)
        tags.extend(keywords[:5])  # Top 5 keywords
        
        return tags
    
    async def _store_classification(
        self,
        user_id: int,
        message_id: str,
        classification: Dict
    ) -> Dict:
        """Store classification in database"""
        
        db_classification = MessageClassification(
            user_id=user_id,
            message_id=message_id,
            **classification
        )
        
        self.db.add(db_classification)
        self.db.commit()
        
        return {**classification, "id": db_classification.id}
    
    async def get_triage_summary(
        self,
        user_id: int,
        time_period: str = "24h"
    ) -> Dict:
        """Get inbox triage summary"""
        
        # Query classifications for period
        classifications = self.db.query(MessageClassification).filter(
            MessageClassification.user_id == user_id,
            MessageClassification.created_at > datetime.now() - self._parse_timedelta(time_period)
        ).all()
        
        return {
            "total_messages": len(classifications),
            "by_category": self._count_by_category(classifications),
            "urgency_distribution": self._urgency_distribution(classifications),
            "top_senders": self._get_top_senders(classifications),
            "spam_detected": sum(1 for c in classifications if c.is_spam)
        }
    
    def _load_or_train_model(self) -> Pipeline:
        """Load pre-trained ML model or train new one"""
        
        # In production, this would load a pre-trained model
        # For now, return a simple pipeline
        return Pipeline([
            ('tfidf', TfidfVectorizer(max_features=1000)),
            ('classifier', RandomForestClassifier(n_estimators=100))
        ])

```

### Pydantic Models

```python
# src/models/schemas.py
from pydantic import BaseModel
from typing import List, Dict, Optional
from datetime import datetime

class MessageAnalysis(BaseModel):
    message_id: str
    category: str
    urgency_score: float
    relevance_score: float
    sender_importance: float
    is_spam: bool
    recommended_action: str
    tags: List[str]
    confidence: float

class TriageSummary(BaseModel):
    total_messages: int
    by_category: Dict[str, int]
    urgency_distribution: Dict[str, int]
    top_senders: List[Dict]
    spam_detected: int
```

### API Routes

```python
# src/app/routes/intelligence.py
from fastapi import APIRouter, Depends, HTTPException, BackgroundTasks
from sqlalchemy.orm import Session
from src.models.schemas import MessageAnalysis, TriageSummary
from src.services.intelligence.inbox_triage import InboxTriageService

router = APIRouter(prefix="/intelligence", tags=["intelligence"])

@router.post("/analyze-message", response_model=MessageAnalysis)
async def analyze_message(
    message_id: str,
    content: str,
    sender: str,
    subject: str,
    db: Session = Depends(get_db),
    current_user = Depends(get_current_user)
):
    """Analyze incoming message for triage"""
    
    service = InboxTriageService(db)
    result = await service.analyze_message(
        user_id=current_user.id,
        message_id=message_id,
        content=content,
        sender=sender,
        subject=subject
    )
    
    return result

@router.get("/triage-summary", response_model=TriageSummary)
async def get_triage_summary(
    period: str = "24h",
    db: Session = Depends(get_db),
    current_user = Depends(get_current_user)
):
    """Get inbox triage summary"""
    
    service = InboxTriageService(db)
    return await service.get_triage_summary(
        user_id=current_user.id,
        time_period=period
    )

@router.post("/train-model", status_code=202)
async def train_triage_model(
    background_tasks: BackgroundTasks,
    db: Session = Depends(get_db),
    current_user = Depends(get_current_user)
):
    """Train/improve triage model with user feedback"""
    
    background_tasks.add_task(
        _train_user_model_task,
        user_id=current_user.id
    )
    
    return {"message": "Training started"}
```

## Component 3.2: Priority Detection Service

### Purpose
Identify and rank high-priority items across all data sources.

```python
# src/services/intelligence/priority_detection.py
from typing import List, Dict
from datetime import datetime
import asyncio

class PriorityDetectionService:
    """Cross-source priority detection"""
    
    def __init__(self, db: Session):
        self.db = db
    
    async def detect_priorities(
        self,
        user_id: int,
        context_window: str = "24h"
    ) -> Dict:
        """Detect priority items across all sources"""
        
        # Gather data from all sources
        tasks_data = await self._get_priority_tasks(user_id, context_window)
        emails_data = await self._get_priority_emails(user_id, context_window)
        calendar_data = await self._get_priority_calendar(user_id, context_window)
        
        # Combine and rank
        all_items = tasks_data + emails_data + calendar_data
        prioritized = await self._rank_items(all_items)
        
        return {
            "priority_items": prioritized[:10],  # Top 10
            "total_items": len(all_items),
            "by_source": self._count_by_source(prioritized),
            "next_action": prioritized[0] if prioritized else None
        }
    
    async def _get_priority_tasks(
        self,
        user_id: int,
        context_window: str
    ) -> List[Dict]:
        """Get high-priority tasks"""
        # Implementation
        pass
    
    async def _rank_items(self, items: List[Dict]) -> List[Dict]:
        """Rank items by priority"""
        # Implementation
        pass
```

## Component 3.3: Calendar Analysis Service

### Purpose
Analyze schedule patterns and provide insights.

```python
# src/services/intelligence/calendar_analysis.py

class CalendarAnalysisService:
    """Calendar insights and optimization"""
    
    async def analyze_schedule(
        self,
        user_id: int,
        period: str = "week"
    ) -> Dict:
        """Analyze calendar patterns"""
        
        events = await self._get_events(user_id, period)
        
        return {
            "busy_times": self._identify_busy_times(events),
            "free_slots": self._find_free_slots(events),
            "meeting_patterns": self._analyze_patterns(events),
            "recommendations": self._generate_recommendations(events),
            "focus_time_availability": self._calculate_focus_time(events)
        }
    
    async def find_meeting_time(
        self,
        user_id: int,
        attendees: List[str],
        duration: int,
        preferences: Dict
    ) -> List[Dict]:
        """Find optimal meeting time"""
        # Implementation
        pass
```

## Component 3.4: Task Prioritization Service

### Purpose
Intelligent task ranking and scheduling.

```python
# src/services/intelligence/task_prioritization.py

class TaskPrioritizationService:
    """Intelligent task prioritization"""
    
    async def prioritize_tasks(
        self,
        user_id: int,
        tasks: List[Dict],
        constraints: Optional[Dict] = None
    ) -> List[Dict]:
        """Prioritize tasks based on multiple factors"""
        
        # Score each task
        scored_tasks = []
        for task in tasks:
            score = await self._calculate_task_score(task, constraints)
            scored_tasks.append({**task, "priority_score": score})
        
        # Sort by score
        sorted_tasks = sorted(scored_tasks, key=lambda x: x["priority_score"], reverse=True)
        
        return sorted_tasks
    
    async def _calculate_task_score(
        self,
        task: Dict,
        constraints: Optional[Dict] = None
    ) -> float:
        """Calculate task priority score"""
        
        score = 0.0
        
        # Urgency factor
        if task.get("due_date"):
            days_until_due = (task["due_date"] - datetime.now()).days
            score += max(0, 1 - (days_until_due / 30))  # Normalize to 30 days
        
        # Importance factor
        importance_map = {"high": 1.0, "medium": 0.6, "low": 0.2}
        score += importance_map.get(task.get("priority", "medium"), 0.6)
        
        # Dependency factor
        if task.get("blocks_other_tasks"):
            score += 0.3
        
        # Time estimation factor
        if constraints and constraints.get("available_time"):
            if task.get("estimated_duration") <= constraints["available_time"]:
                score += 0.2
        
        return min(score, 1.0)
```

## Component 3.5: Daily Briefing Service

### Purpose
Generate personalized daily summaries with insights and recommendations.

```python
# src/services/intelligence/briefing.py

class BriefingService:
    """Daily briefing generation"""
    
    async def generate_briefing(
        self,
        user_id: int,
        format: str = "html",
        sections: Optional[List[str]] = None
    ) -> str:
        """Generate personalized daily briefing"""
        
        sections = sections or [
            "priorities", "schedule", "messages", "recommendations"
        ]
        
        briefing_data = {}
        
        for section in sections:
            if section == "priorities":
                briefing_data["priorities"] = await self._get_priority_section(user_id)
            elif section == "schedule":
                briefing_data["schedule"] = await self._get_schedule_section(user_id)
            elif section == "messages":
                briefing_data["messages"] = await self._get_messages_section(user_id)
            elif section == "recommendations":
                briefing_data["recommendations"] = await self._get_recommendations_section(user_id)
        
        # Format briefing
        if format == "html":
            return self._format_html_briefing(briefing_data)
        elif format == "text":
            return self._format_text_briefing(briefing_data)
        elif format == "pdf":
            return await self._format_pdf_briefing(briefing_data)
    
    async def schedule_briefing(
        self,
        user_id: int,
        time: str = "09:00",
        frequency: str = "daily"
    ) -> Dict:
        """Schedule daily briefing delivery"""
        # Implementation
        pass
```

## Testing Strategy

### Unit Tests

```python
# tests/unit/test_inbox_triage.py
import pytest
from src.services.intelligence.inbox_triage import InboxTriageService

class TestInboxTriageService:
    
    @pytest.fixture
    def service(self, db):
        return InboxTriageService(db)
    
    @pytest.mark.asyncio
    async def test_analyze_message_urgent(self, service):
        result = await service.analyze_message(
            user_id=1,
            message_id="msg_123",
            content="URGENT: Please review this immediately!",
            sender="boss@company.com",
            subject="Critical Issue"
        )
        
        assert result["urgency_score"] > 0.7
        assert result["category"] in ["urgent", "important"]
    
    @pytest.mark.asyncio
    async def test_spam_detection(self, service):
        result = await service.analyze_message(
            user_id=1,
            message_id="msg_456",
            content="CLICK HERE NOW! Limited time offer! Verify your account!",
            sender="spam@example.com",
            subject="You won!"
        )
        
        assert result["is_spam"] is True
```

---

## Phase 3 Implementation Checklist

- [ ] Inbox Triage Service
  - [ ] Feature extraction
  - [ ] Urgency scoring
  - [ ] Relevance scoring
  - [ ] Spam detection
  - [ ] Category prediction
  - [ ] Action recommendation
  - [ ] Tag generation
  - [ ] Database schema
  - [ ] Unit tests (>80%)
  - [ ] Integration tests

- [ ] Priority Detection Service
  - [ ] Cross-source analysis
  - [ ] Item ranking
  - [ ] Next action identification
  - [ ] Unit tests
  - [ ] Integration tests

- [ ] Calendar Analysis Service
  - [ ] Pattern analysis
  - [ ] Focus time calculation
  - [ ] Meeting time optimization
  - [ ] Unit tests
  - [ ] Integration tests

- [ ] Task Prioritization Service
  - [ ] Multi-factor scoring
  - [ ] Dependency analysis
  - [ ] Constraint handling
  - [ ] Unit tests
  - [ ] Integration tests

- [ ] Daily Briefing Service
  - [ ] Section generation
  - [ ] Formatting (HTML/PDF/Text)
  - [ ] Scheduling
  - [ ] Unit tests
  - [ ] Integration tests

---

**Last Updated:** September 8, 2026  
**Status:** Ready for Implementation
