# Phase 4: Safety - Implementation Guide

## Overview

Phase 4 implements comprehensive security, privacy, and compliance controls to ensure the AI Assistant operates safely with proper authorization, data protection, and user oversight.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│    Phase 3: Intelligence Layer (Analysis & Insights) │
└────────────────────────┬────────────────────────────┘
                         │
        ┌────────────────┴────────────────┐
        │                                 │
        ▼                                 ▼
┌──────────────────┐            ┌──────────────────┐
│   Action Handler │            │   Query Engine   │
│   (from Phase 3) │            │   (from Phase 3) │
└────────┬─────────┘            └────────┬─────────┘
         │                               │
         └───────────────┬───────────────┘
                         │
        ┌────────────────▼────────────────┐
        │   Safety & Authorization Layer   │
        │   (Phase 4 - Core)              │
        └────────────────┬────────────────┘
                         │
        ┌────────────────┼────────────────────┐
        │                │                    │
        ▼                ▼                    ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Privacy      │  │ Access       │  │ Confirmation │
│ Rules Engine │  │ Control      │  │ Workflow     │
│ (4.1)        │  │ (4.2)        │  │ (4.3)        │
└──────────────┘  └──────────────┘  └──────────────┘
        │                │                    │
        └────────────────┼────────────────────┘
                         │
        ┌────────────────▼────────────────┐
        │  Data Protection Layer (4.4-4.5)│
        │  - Encryption                   │
        │  - Data Separation              │
        │  - Information Masking          │
        └────────────────┬────────────────┘
                         │
        ┌────────────────▼────────────────┐
        │      Audit & Logging Layer      │
        │  - Access logging               │
        │  - Change tracking              │
        │  - Compliance reporting         │
        └────────────────┬────────────────┘
                         │
        ┌────────────────▼────────────────┐
        │      Database & Storage         │
        │  (PostgreSQL, Encrypted Data)   │
        └─────────────────────────────────┘
```

## Component 4.1: Privacy Rules Engine

### Purpose
Define, manage, and enforce data access and usage policies.

### Database Schema

```sql
-- Privacy rules definition
CREATE TABLE privacy_rules (
    id SERIAL PRIMARY KEY,
    organization_id INTEGER REFERENCES organizations(id),
    name VARCHAR(255) UNIQUE NOT NULL,
    description TEXT,
    version INTEGER DEFAULT 1,
    is_active BOOLEAN DEFAULT true,
    rule_type VARCHAR(50), -- 'access', 'data_handling', 'retention', 'sharing'
    definition JSONB NOT NULL, -- Rule definition
    created_by INTEGER REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    published_at TIMESTAMP
);

-- Rule conditions and actions
CREATE TABLE rule_conditions (
    id SERIAL PRIMARY KEY,
    rule_id INTEGER REFERENCES privacy_rules(id),
    condition_type VARCHAR(100), -- 'data_type', 'user_role', 'time_based', 'location_based'
    operator VARCHAR(50), -- 'equals', 'contains', 'greater_than', 'between'
    value JSONB,
    sequence INTEGER
);

CREATE TABLE rule_actions (
    id SERIAL PRIMARY KEY,
    rule_id INTEGER REFERENCES privacy_rules(id),
    action_type VARCHAR(100), -- 'allow', 'deny', 'require_approval', 'log', 'encrypt'
    action_params JSONB,
    sequence INTEGER
);

-- Rule evaluation results
CREATE TABLE rule_evaluations (
    id SERIAL PRIMARY KEY,
    rule_id INTEGER REFERENCES privacy_rules(id),
    user_id INTEGER REFERENCES users(id),
    action VARCHAR(255),
    resource_type VARCHAR(100),
    resource_id VARCHAR(255),
    result VARCHAR(50), -- 'allowed', 'denied', 'requires_approval'
    reasoning TEXT,
    evaluated_at TIMESTAMP DEFAULT NOW()
);

-- Policy versions and history
CREATE TABLE policy_history (
    id SERIAL PRIMARY KEY,
    rule_id INTEGER REFERENCES privacy_rules(id),
    version INTEGER,
    previous_definition JSONB,
    new_definition JSONB,
    change_reason TEXT,
    changed_by INTEGER REFERENCES users(id),
    changed_at TIMESTAMP DEFAULT NOW()
);
```

### Service Implementation

```python
# src/services/safety/privacy_rules.py
from typing import Dict, List, Optional, Any
from enum import Enum
import logging
from datetime import datetime
from sqlalchemy.orm import Session
from sqlalchemy import and_, or_

logger = logging.getLogger(__name__)

class RuleType(str, Enum):
    ACCESS = "access"
    DATA_HANDLING = "data_handling"
    RETENTION = "retention"
    SHARING = "sharing"

class PrivacyRulesEngine:
    """Privacy rules definition and enforcement"""
    
    def __init__(self, db: Session):
        self.db = db
    
    async def create_rule(
        self,
        organization_id: int,
        name: str,
        rule_type: RuleType,
        definition: Dict,
        description: Optional[str] = None,
        created_by: int = None
    ) -> Dict:
        """Create a new privacy rule"""
        
        try:
            # Validate rule definition
            await self._validate_rule_definition(definition, rule_type)
            
            # Create rule
            rule = PrivacyRule(
                organization_id=organization_id,
                name=name,
                rule_type=rule_type.value,
                definition=definition,
                description=description,
                created_by=created_by
            )
            
            self.db.add(rule)
            self.db.commit()
            
            logger.info(f"Privacy rule created: {name} (ID: {rule.id})")
            
            return {
                "id": rule.id,
                "name": rule.name,
                "rule_type": rule.rule_type,
                "created_at": rule.created_at,
                "is_active": rule.is_active
            }
        
        except Exception as e:
            logger.error(f"Failed to create privacy rule: {str(e)}")
            self.db.rollback()
            raise
    
    async def evaluate_rule(
        self,
        rule_id: int,
        user_id: int,
        action: str,
        resource_type: str,
        resource_id: str,
        context: Optional[Dict] = None
    ) -> Dict:
        """Evaluate if user can perform action on resource"""
        
        try:
            # Fetch rule
            rule = self.db.query(PrivacyRule).filter(
                PrivacyRule.id == rule_id,
                PrivacyRule.is_active == True
            ).first()
            
            if not rule:
                return {
                    "rule_id": rule_id,
                    "result": "not_found",
                    "allowed": False,
                    "reasoning": "Rule not found or inactive"
                }
            
            # Get user details
            user = self.db.query(User).filter(User.id == user_id).first()
            
            # Build evaluation context
            eval_context = {
                "user_id": user_id,
                "user_roles": [r.name for r in user.roles],
                "action": action,
                "resource_type": resource_type,
                "resource_id": resource_id,
                "timestamp": datetime.now(),
                **(context or {})
            }
            
            # Evaluate conditions
            conditions_met = await self._evaluate_conditions(
                rule.definition.get("conditions", []),
                eval_context
            )
            
            # Apply actions
            result, reasoning = await self._apply_actions(
                rule.definition.get("actions", []),
                conditions_met,
                eval_context
            )
            
            # Log evaluation
            evaluation = RuleEvaluation(
                rule_id=rule_id,
                user_id=user_id,
                action=action,
                resource_type=resource_type,
                resource_id=resource_id,
                result=result,
                reasoning=reasoning
            )
            self.db.add(evaluation)
            self.db.commit()
            
            return {
                "rule_id": rule_id,
                "result": result,
                "allowed": result == "allowed",
                "reasoning": reasoning,
                "evaluation_id": evaluation.id
            }
        
        except Exception as e:
            logger.error(f"Rule evaluation failed: {str(e)}")
            raise
    
    async def _evaluate_conditions(
        self,
        conditions: List[Dict],
        context: Dict
    ) -> bool:
        """Evaluate all conditions"""
        
        # All conditions must be true (AND logic)
        for condition in conditions:
            if not await self._evaluate_single_condition(condition, context):
                return False
        
        return True
    
    async def _evaluate_single_condition(
        self,
        condition: Dict,
        context: Dict
    ) -> bool:
        """Evaluate a single condition"""
        
        condition_type = condition.get("type")
        operator = condition.get("operator")
        value = condition.get("value")
        
        if condition_type == "data_type":
            return await self._check_data_type_condition(
                operator, value, context
            )
        elif condition_type == "user_role":
            return await self._check_role_condition(
                operator, value, context
            )
        elif condition_type == "time_based":
            return await self._check_time_condition(
                operator, value, context
            )
        elif condition_type == "location_based":
            return await self._check_location_condition(
                operator, value, context
            )
        
        return False
    
    async def _check_data_type_condition(
        self,
        operator: str,
        value: Any,
        context: Dict
    ) -> bool:
        """Check data type condition"""
        
        resource_type = context.get("resource_type")
        
        if operator == "equals":
            return resource_type == value
        elif operator == "in":
            return resource_type in value
        elif operator == "contains":
            return value in resource_type
        
        return False
    
    async def _check_role_condition(
        self,
        operator: str,
        value: Any,
        context: Dict
    ) -> bool:
        """Check role condition"""
        
        user_roles = context.get("user_roles", [])
        
        if operator == "has_role":
            return value in user_roles
        elif operator == "has_any_role":
            return any(role in user_roles for role in value)
        elif operator == "has_all_roles":
            return all(role in user_roles for role in value)
        elif operator == "not_has_role":
            return value not in user_roles
        
        return False
    
    async def _check_time_condition(
        self,
        operator: str,
        value: Any,
        context: Dict
    ) -> bool:
        """Check time-based condition"""
        
        current_time = context.get("timestamp", datetime.now())
        
        if operator == "business_hours":
            return 9 <= current_time.hour < 17 and current_time.weekday() < 5
        elif operator == "after_hours":
            return not (9 <= current_time.hour < 17 and current_time.weekday() < 5)
        elif operator == "in_range":
            start, end = value["start"], value["end"]
            return start <= current_time.hour < end
        
        return False
    
    async def _check_location_condition(
        self,
        operator: str,
        value: Any,
        context: Dict
    ) -> bool:
        """Check location condition"""
        
        user_ip = context.get("ip_address")
        
        if operator == "in_office":
            # Check against office IP ranges
            return self._is_office_ip(user_ip)
        elif operator == "from_trusted_network":
            return self._is_trusted_network(user_ip)
        
        return False
    
    async def _apply_actions(
        self,
        actions: List[Dict],
        conditions_met: bool,
        context: Dict
    ) -> tuple:
        """Apply rule actions"""
        
        if not conditions_met:
            return "denied", "Conditions not met"
        
        for action in actions:
            action_type = action.get("type")
            
            if action_type == "allow":
                return "allowed", "Allowed by privacy rule"
            elif action_type == "deny":
                return "denied", action.get("reason", "Denied by privacy rule")
            elif action_type == "require_approval":
                return "requires_approval", "Approval required"
            elif action_type == "log":
                await self._log_access(context, action)
            elif action_type == "encrypt":
                context["apply_encryption"] = True
        
        return "allowed", "Passed privacy evaluation"
    
    async def _validate_rule_definition(
        self,
        definition: Dict,
        rule_type: RuleType
    ) -> bool:
        """Validate rule definition structure"""
        
        required_fields = ["conditions", "actions"]
        
        for field in required_fields:
            if field not in definition:
                raise ValueError(f"Missing required field: {field}")
        
        # Validate each condition
        for condition in definition.get("conditions", []):
            required_condition_fields = ["type", "operator", "value"]
            if not all(f in condition for f in required_condition_fields):
                raise ValueError("Invalid condition structure")
        
        # Validate each action
        for action in definition.get("actions", []):
            if "type" not in action:
                raise ValueError("Invalid action structure")
        
        return True
    
    async def get_applicable_rules(
        self,
        organization_id: int,
        rule_type: Optional[RuleType] = None
    ) -> List[Dict]:
        """Get all applicable rules for organization"""
        
        query = self.db.query(PrivacyRule).filter(
            PrivacyRule.organization_id == organization_id,
            PrivacyRule.is_active == True
        )
        
        if rule_type:
            query = query.filter(PrivacyRule.rule_type == rule_type.value)
        
        rules = query.all()
        
        return [
            {
                "id": r.id,
                "name": r.name,
                "rule_type": r.rule_type,
                "description": r.description
            }
            for r in rules
        ]
    
    async def audit_rule_access(
        self,
        organization_id: int,
        days: int = 30
    ) -> Dict:
        """Get audit report of rule evaluations"""
        
        evaluations = self.db.query(RuleEvaluation).join(
            PrivacyRule
        ).filter(
            PrivacyRule.organization_id == organization_id,
            RuleEvaluation.evaluated_at > datetime.now() - timedelta(days=days)
        ).all()
        
        return {
            "total_evaluations": len(evaluations),
            "allowed": sum(1 for e in evaluations if e.result == "allowed"),
            "denied": sum(1 for e in evaluations if e.result == "denied"),
            "requires_approval": sum(1 for e in evaluations if e.result == "requires_approval"),
            "by_resource_type": self._group_by_resource_type(evaluations),
            "by_user": self._group_by_user(evaluations)
        }
```

### API Routes

```python
# src/app/routes/safety.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from src.models.schemas import PrivacyRuleCreate, RuleEvaluationResponse
from src.services.safety.privacy_rules import PrivacyRulesEngine

router = APIRouter(prefix="/safety", tags=["safety"])

@router.post("/rules", response_model=Dict)
async def create_privacy_rule(
    rule_data: PrivacyRuleCreate,
    db: Session = Depends(get_db),
    current_user = Depends(get_current_user)
):
    """Create a new privacy rule"""
    
    engine = PrivacyRulesEngine(db)
    return await engine.create_rule(
        organization_id=current_user.organization_id,
        name=rule_data.name,
        rule_type=rule_data.rule_type,
        definition=rule_data.definition,
        created_by=current_user.id
    )

@router.post("/evaluate-rule")
async def evaluate_rule(
    rule_id: int,
    action: str,
    resource_type: str,
    resource_id: str,
    db: Session = Depends(get_db),
    current_user = Depends(get_current_user)
):
    """Evaluate privacy rule"""
    
    engine = PrivacyRulesEngine(db)
    return await engine.evaluate_rule(
        rule_id=rule_id,
        user_id=current_user.id,
        action=action,
        resource_type=resource_type,
        resource_id=resource_id
    )

@router.get("/rules", response_model=List[Dict])
async def get_privacy_rules(
    rule_type: Optional[str] = None,
    db: Session = Depends(get_db),
    current_user = Depends(get_current_user)
):
    """Get all privacy rules"""
    
    engine = PrivacyRulesEngine(db)
    return await engine.get_applicable_rules(
        organization_id=current_user.organization_id,
        rule_type=rule_type
    )

@router.get("/audit-report")
async def get_audit_report(
    days: int = 30,
    db: Session = Depends(get_db),
    current_user = Depends(get_current_user)
):
    """Get audit report"""
    
    engine = PrivacyRulesEngine(db)
    return await engine.audit_rule_access(
        organization_id=current_user.organization_id,
        days=days
    )
```

## Component 4.2: Access Control Service

### Purpose
Implement Role-Based Access Control (RBAC) and Attribute-Based Access Control (ABAC).

```python
# src/services/safety/access_control.py
from typing import Dict, List, Optional
from enum import Enum
import logging

logger = logging.getLogger(__name__)

class Permission(str, Enum):
    READ = "read"
    CREATE = "create"
    UPDATE = "update"
    DELETE = "delete"
    EXECUTE = "execute"
    SHARE = "share"

class AccessControlService:
    """Access control enforcement"""
    
    def __init__(self, db: Session):
        self.db = db
    
    async def check_permission(
        self,
        user_id: int,
        action: str,
        resource_type: str,
        resource_id: Optional[str] = None,
        context: Optional[Dict] = None
    ) -> Dict:
        """Check if user has permission for action"""
        
        try:
            # Get user roles
            user = self.db.query(User).filter(User.id == user_id).first()
            user_roles = [r.name for r in user.roles]
            
            # Check RBAC
            rbac_result = await self._check_rbac(
                user_roles, action, resource_type
            )
            
            if not rbac_result["allowed"]:
                return {
                    "allowed": False,
                    "reason": "Insufficient role permissions",
                    "method": "RBAC"
                }
            
            # Check ABAC
            abac_result = await self._check_abac(
                user_id, action, resource_type, resource_id, context
            )
            
            if not abac_result["allowed"]:
                return {
                    "allowed": False,
                    "reason": abac_result["reason"],
                    "method": "ABAC"
                }
            
            # Log access check
            await self._log_access_check(
                user_id, action, resource_type, resource_id, True
            )
            
            return {
                "allowed": True,
                "user_id": user_id,
                "action": action,
                "resource_type": resource_type,
                "method": "RBAC+ABAC"
            }
        
        except Exception as e:
            logger.error(f"Access check failed: {str(e)}")
            return {"allowed": False, "reason": str(e)}
    
    async def _check_rbac(
        self,
        user_roles: List[str],
        action: str,
        resource_type: str
    ) -> Dict:
        """Check role-based access"""
        
        # Query permissions for user's roles
        permissions = self.db.query(RolePermission).join(
            Role
        ).filter(
            Role.name.in_(user_roles),
            RolePermission.action == action,
            RolePermission.resource_type == resource_type
        ).all()
        
        return {
            "allowed": len(permissions) > 0,
            "permissions": [p.name for p in permissions]
        }
    
    async def _check_abac(
        self,
        user_id: int,
        action: str,
        resource_type: str,
        resource_id: Optional[str],
        context: Optional[Dict]
    ) -> Dict:
        """Check attribute-based access"""
        
        # Get resource
        resource = await self._get_resource(resource_type, resource_id)
        
        if not resource:
            return {"allowed": True}  # Resource doesn't exist, allow
        
        # Check resource-specific rules
        if resource_type == "health_record":
            return await self._check_health_record_access(
                user_id, resource, action, context
            )
        elif resource_type == "task":
            return await self._check_task_access(
                user_id, resource, action, context
            )
        
        return {"allowed": True}
    
    async def _check_health_record_access(
        self,
        user_id: int,
        resource: Dict,
        action: str,
        context: Optional[Dict]
    ) -> Dict:
        """Health record specific access control"""
        
        # Owner can do anything
        if resource.get("owner_id") == user_id:
            return {"allowed": True}
        
        # Check delegation
        is_delegated = self.db.query(DataDelegation).filter(
            DataDelegation.resource_id == resource.get("id"),
            DataDelegation.delegated_to == user_id,
            DataDelegation.is_active == True
        ).first()
        
        if is_delegated:
            # Check delegated permissions
            allowed_actions = is_delegated.allowed_actions
            return {"allowed": action in allowed_actions}
        
        return {"allowed": False, "reason": "Access denied"}
    
    async def require_permission(
        self,
        user_id: int,
        action: str,
        resource_type: str,
        resource_id: Optional[str] = None,
        context: Optional[Dict] = None
    ) -> bool:
        """Require permission or raise exception"""
        
        result = await self.check_permission(
            user_id, action, resource_type, resource_id, context
        )
        
        if not result["allowed"]:
            raise UnauthorizedAccessError(result["reason"])
        
        return True
```

## Component 4.3: Confirmation Workflow

### Purpose
Require user confirmation for sensitive operations.

```python
# src/services/safety/confirmation.py
from typing import Dict, List
from enum import Enum
import secrets
import logging

logger = logging.getLogger(__name__)

class ConfirmationSeverity(str, Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"

class ConfirmationService:
    """Confirmation workflow for sensitive operations"""
    
    def __init__(self, db: Session):
        self.db = db
    
    async def request_confirmation(
        self,
        user_id: int,
        action: str,
        resource_type: str,
        severity: ConfirmationSeverity = ConfirmationSeverity.MEDIUM,
        required_factors: Optional[List[str]] = None,
        metadata: Optional[Dict] = None
    ) -> Dict:
        """Request confirmation for sensitive action"""
        
        # Determine required factors based on severity
        if not required_factors:
            required_factors = self._get_default_factors(severity)
        
        # Create confirmation request
        confirmation_code = secrets.token_urlsafe(32)
        
        confirmation = ConfirmationRequest(
            user_id=user_id,
            action=action,
            resource_type=resource_type,
            severity=severity.value,
            required_factors=required_factors,
            confirmation_code=confirmation_code,
            metadata=metadata,
            expires_at=datetime.now() + timedelta(minutes=15)
        )
        
        self.db.add(confirmation)
        self.db.commit()
        
        # Send confirmation requests to user
        await self._send_confirmation_requests(user_id, confirmation, required_factors)
        
        logger.info(f"Confirmation requested for {action} by user {user_id}")
        
        return {
            "confirmation_id": confirmation.id,
            "required_factors": required_factors,
            "expires_at": confirmation.expires_at
        }
    
    async def verify_confirmation(
        self,
        confirmation_id: int,
        verification_codes: Dict[str, str]
    ) -> Dict:
        """Verify confirmation factors"""
        
        confirmation = self.db.query(ConfirmationRequest).filter(
            ConfirmationRequest.id == confirmation_id
        ).first()
        
        if not confirmation:
            raise ConfirmationNotFoundError()
        
        if confirmation.expires_at < datetime.now():
            raise ConfirmationExpiredError()
        
        if confirmation.is_verified:
            raise ConfirmationAlreadyUsedError()
        
        # Verify each required factor
        results = {}
        for factor in confirmation.required_factors:
            if factor not in verification_codes:
                raise MissingVerificationFactorError(factor)
            
            verified = await self._verify_factor(factor, verification_codes[factor])
            results[factor] = verified
        
        # All factors must be verified
        if not all(results.values()):
            confirmation.failed_attempts += 1
            self.db.commit()
            
            if confirmation.failed_attempts >= 3:
                confirmation.is_locked = True
                self.db.commit()
            
            raise VerificationFailedError()
        
        # Mark as verified
        confirmation.is_verified = True
        confirmation.verified_at = datetime.now()
        self.db.commit()
        
        logger.info(f"Confirmation {confirmation_id} verified successfully")
        
        return {
            "confirmation_id": confirmation_id,
            "verified": True,
            "action": confirmation.action,
            "verified_at": confirmation.verified_at
        }
    
    async def _verify_factor(self, factor: str, code: str) -> bool:
        """Verify individual confirmation factor"""
        
        if factor == "password":
            return await self._verify_password(code)
        elif factor == "2fa":
            return await self._verify_2fa(code)
        elif factor == "email":
            return await self._verify_email_code(code)
        elif factor == "sms":
            return await self._verify_sms_code(code)
        elif factor == "biometric":
            return await self._verify_biometric(code)
        
        return False
    
    async def _get_default_factors(self, severity: ConfirmationSeverity) -> List[str]:
        """Get default confirmation factors for severity level"""
        
        factors = {
            ConfirmationSeverity.LOW: ["password"],
            ConfirmationSeverity.MEDIUM: ["password", "email"],
            ConfirmationSeverity.HIGH: ["password", "2fa"],
            ConfirmationSeverity.CRITICAL: ["password", "2fa", "email"]
        }
        
        return factors.get(severity, ["password"])
```

## Component 4.4: Data Separation Service

### Purpose
Enforce work/personal data boundaries and isolation.

```python
# src/services/safety/data_separation.py
from typing import List, Dict
from enum import Enum
import logging

logger = logging.getLogger(__name__)

class DataClassification(str, Enum):
    PERSONAL_HEALTH = "personal_health"
    PERSONAL_WORK = "personal_work"
    BUSINESS_CONFIDENTIAL = "business_confidential"
    PUBLIC = "public"

class AccessLevel(str, Enum):
    OWNER_ONLY = "owner_only"
    ORGANIZATION = "organization"
    TEAM = "team"
    PUBLIC = "public"

class DataSeparationService:
    """Work/personal data separation"""
    
    def __init__(self, db: Session):
        self.db = db
    
    async def classify_data(
        self,
        resource_id: str,
        resource_type: str,
        classification: DataClassification,
        access_level: AccessLevel,
        user_id: int
    ) -> Dict:
        """Classify data item"""
        
        classification_record = DataClassification(
            resource_id=resource_id,
            resource_type=resource_type,
            classification=classification.value,
            access_level=access_level.value,
            created_by=user_id,
            created_at=datetime.now()
        )
        
        self.db.add(classification_record)
        self.db.commit()
        
        logger.info(f"Data classified: {resource_id} as {classification.value}")
        
        return {
            "resource_id": resource_id,
            "classification": classification.value,
            "access_level": access_level.value
        }
    
    async def get_user_data(
        self,
        user_id: int,
        workspace: str,
        include_work: bool = True,
        include_personal: bool = True
    ) -> List[Dict]:
        """Get user data filtered by workspace"""
        
        # Build classification filter
        allowed_classifications = []
        
        if include_work:
            allowed_classifications.extend([
                DataClassification.PERSONAL_WORK.value,
                DataClassification.BUSINESS_CONFIDENTIAL.value
            ])
        
        if include_personal:
            allowed_classifications.extend([
                DataClassification.PERSONAL_HEALTH.value
            ])
        
        # Query data with classification
        data = self.db.query(DataItem).join(
            DataClassificationRecord
        ).filter(
            DataItem.owner_id == user_id,
            DataClassificationRecord.classification.in_(allowed_classifications)
        ).all()
        
        return [self._format_data_item(item) for item in data]
    
    async def enforce_separation(
        self,
        user_id: int,
        action: str,
        resource_id: str
    ) -> bool:
        """Enforce data separation rules"""
        
        # Get resource classification
        classification = self.db.query(DataClassificationRecord).filter(
            DataClassificationRecord.resource_id == resource_id
        ).first()
        
        if not classification:
            return True  # Not classified, allow
        
        # Check if action violates separation
        if classification.classification == DataClassification.PERSONAL_HEALTH.value:
            # Personal health data cannot be shared with work context
            if action == "share_with_organization":
                return False
        
        return True
```

## Component 4.5: Information Protection Service

### Purpose
Encrypt, mask, and protect sensitive information.

```python
# src/services/safety/information_protection.py
from cryptography.fernet import Fernet
from typing import Any, Dict
import logging

logger = logging.getLogger(__name__)

class InformationProtectionService:
    """Data encryption and masking"""
    
    def __init__(self, encryption_key: str = None):
        self.encryption_key = encryption_key or os.getenv("DATA_ENCRYPTION_KEY")
        self.cipher = Fernet(self.encryption_key.encode())
    
    async def encrypt_sensitive_data(
        self,
        data: Any,
        data_type: str,
        user_id: int
    ) -> str:
        """Encrypt sensitive data"""
        
        try:
            # Serialize data
            serialized = json.dumps(data)
            
            # Encrypt
            encrypted = self.cipher.encrypt(serialized.encode())
            
            # Log encryption
            await self._log_encryption(user_id, data_type, encrypted.decode())
            
            logger.info(f"Data encrypted for user {user_id}")
            
            return encrypted.decode()
        
        except Exception as e:
            logger.error(f"Encryption failed: {str(e)}")
            raise
    
    async def decrypt_and_log_access(
        self,
        encrypted_data: str,
        user_id: int,
        accessor_id: int,
        reason: str
    ) -> Any:
        """Decrypt with access logging"""
        
        try:
            # Log access attempt
            access_log = AccessLog(
                resource_id=encrypted_data[:50],  # Reference
                user_id=user_id,
                accessor_id=accessor_id,
                action="decrypt",
                reason=reason,
                accessed_at=datetime.now()
            )
            self.db.add(access_log)
            self.db.commit()
            
            # Decrypt
            decrypted = self.cipher.decrypt(encrypted_data.encode())
            data = json.loads(decrypted.decode())
            
            logger.info(f"Data decrypted for user {accessor_id} with reason: {reason}")
            
            return data
        
        except Exception as e:
            logger.error(f"Decryption failed: {str(e)}")
            raise
    
    async def mask_sensitive_fields(
        self,
        data: Dict,
        sensitive_fields: List[str]
    ) -> Dict:
        """Mask sensitive fields in data"""
        
        masked = data.copy()
        
        for field in sensitive_fields:
            if field in masked:
                value = masked[field]
                if isinstance(value, str):
                    # Show only first and last character
                    masked[field] = value[0] + "*" * (len(value) - 2) + value[-1]
                elif isinstance(value, (int, float)):
                    masked[field] = "***"
        
        return masked
    
    async def get_access_audit_trail(
        self,
        resource_id: str,
        days: int = 30
    ) -> List[Dict]:
        """Get audit trail of who accessed data"""
        
        logs = self.db.query(AccessLog).filter(
            AccessLog.resource_id == resource_id,
            AccessLog.accessed_at > datetime.now() - timedelta(days=days)
        ).all()
        
        return [
            {
                "accessor_id": log.accessor_id,
                "accessed_at": log.accessed_at,
                "reason": log.reason
            }
            for log in logs
        ]
```

## Testing Strategy

### Unit Tests for Privacy Rules

```python
# tests/unit/test_privacy_rules.py
import pytest
from src.services.safety.privacy_rules import PrivacyRulesEngine

class TestPrivacyRulesEngine:
    
    @pytest.fixture
    def engine(self, db):
        return PrivacyRulesEngine(db)
    
    @pytest.mark.asyncio
    async def test_create_rule(self, engine):
        rule_def = {
            "conditions": [
                {"type": "user_role", "operator": "has_role", "value": "doctor"}
            ],
            "actions": [
                {"type": "allow"}
            ]
        }
        
        result = await engine.create_rule(
            organization_id=1,
            name="Doctors Access Health Records",
            rule_type="access",
            definition=rule_def
        )
        
        assert result["id"] is not None
        assert result["name"] == "Doctors Access Health Records"
    
    @pytest.mark.asyncio
    async def test_evaluate_rule_allowed(self, engine, db):
        # Create rule
        rule_def = {
            "conditions": [
                {"type": "user_role", "operator": "has_role", "value": "doctor"}
            ],
            "actions": [{"type": "allow"}]
        }
        
        rule = await engine.create_rule(
            organization_id=1,
            name="Test Rule",
            rule_type="access",
            definition=rule_def
        )
        
        # User with doctor role
        result = await engine.evaluate_rule(
            rule_id=rule["id"],
            user_id=1,
            action="read",
            resource_type="health_record",
            resource_id="rec_123",
            context={"user_roles": ["doctor"]}
        )
        
        assert result["allowed"] is True
```

## Phase 4 Implementation Checklist

- [ ] Privacy Rules Engine
  - [ ] Rule definition model
  - [ ] Condition evaluation
  - [ ] Action application
  - [ ] Rule versioning
  - [ ] Audit logging
  - [ ] Unit tests >80%
  - [ ] Integration tests

- [ ] Access Control Service
  - [ ] RBAC implementation
  - [ ] ABAC implementation
  - [ ] Permission checking
  - [ ] Role inheritance
  - [ ] Unit tests >80%
  - [ ] Integration tests

- [ ] Confirmation Workflow
  - [ ] Multi-factor verification
  - [ ] Confirmation expiry
  - [ ] Rate limiting
  - [ ] Email notifications
  - [ ] Unit tests >80%
  - [ ] Integration tests

- [ ] Data Separation
  - [ ] Classification system
  - [ ] Access level enforcement
  - [ ] Workspace isolation
  - [ ] Unit tests >80%
  - [ ] Integration tests

- [ ] Information Protection
  - [ ] Encryption at rest
  - [ ] Encryption in transit
  - [ ] Data masking
  - [ ] Access audit trail
  - [ ] Key rotation
  - [ ] Unit tests >80%
  - [ ] Integration tests

- [ ] Security Audit
  - [ ] Penetration testing
  - [ ] Code review
  - [ ] Vulnerability scanning
  - [ ] Compliance verification

---

## Security Compliance Checklist

- [ ] GDPR compliance
- [ ] HIPAA compliance (if handling health data)
- [ ] SOC 2 controls
- [ ] PCI DSS (if handling payments)
- [ ] Data protection regulations
- [ ] Privacy policy
- [ ] Terms of service
- [ ] Incident response plan

---

**Last Updated:** September 8, 2026  
**Status:** Ready for Implementation
