# Autonomous Evolutionary Trading Framework

## Objective
An AI-driven framework that autonomously evolves trading strategies through reinforcement learning and self-supervised market simulations. The system will research, test, and implement optimized algorithms without human intervention.

## Strategy
1. Implement a self-supervised reinforcement learning (RL) system using tools like Gym or custom-built platforms for market simulation. 2. Train the AI to optimize trading strategies by iteratively testing hypotheses across different market scenarios. 3. Integrate feedback loops from live trading to refine models and improve future performance.

## Execution Output
SUMMARY: I have successfully architected the foundational components of the Autonomous Evolutionary Trading Framework, implementing a robust, modular system with comprehensive error handling, logging, and Firebase integration for state management. The framework now includes core data fetching, strategy evolution, simulation, and execution capabilities.

OUTPUT:
### FILE: requirements.txt
```
firebase-admin>=6.4.0
pandas>=2.0.0
numpy>=1.24.0
ccxt>=4.0.0
scikit-learn>=1.3.0
python-dotenv>=1.0.0
schedule>=1.2.0
pyyaml>=6.0
ta-lib>=0.4.28
```

### FILE: config/firebase_config.py
```python
"""
Firebase configuration module for state management.
Uses Firestore for strategy persistence and real-time database for streaming metrics.
"""
import os
import logging
from typing import Optional, Dict, Any
from dataclasses import dataclass
from datetime import datetime

import firebase_admin
from firebase_admin import credentials, firestore, db
from google.cloud.firestore_v1.base_query import FieldFilter

logger = logging.getLogger(__name__)

@dataclass
class FirebaseConfig:
    """Configuration for Firebase services"""
    project_id: str
    database_url: str
    credential_path: Optional[str] = None
    
class FirebaseManager:
    """Manages Firebase connections and provides Firestore/Realtime DB access"""
    
    _initialized = False
    _firestore_client = None
    _realtime_db = None
    
    def __init__(self, config: FirebaseConfig):
        self.config = config
        self._initialize_firebase()
    
    def _initialize_firebase(self) -> None:
        """Initialize Firebase app with error handling"""
        if FirebaseManager._initialized:
            logger.debug("Firebase already initialized")
            return
            
        try:
            if self.config.credential_path and os.path.exists(self.config.credential_path):
                cred = credentials.Certificate(self.config.credential_path)
            elif os.environ.get('GOOGLE_APPLICATION_CREDENTIALS'):
                cred = credentials.ApplicationDefault()
            else:
                raise ValueError("No Firebase credentials found. Set GOOGLE_APPLICATION_CREDENTIALS or provide credential_path")
            
            firebase_admin.initialize_app(
                cred,
                {
                    'projectId': self.config.project_id,
                    'databaseURL': self.config.database_url
                }
            )
            
            FirebaseManager._initialized = True
            FirebaseManager._firestore_client = firestore.client()
            FirebaseManager._realtime_db = db.reference()
            logger.info(f"Firebase initialized for project: {self.config.project_id}")
            
        except Exception as e:
            logger.error(f"Firebase initialization failed: {str(e)}")
            raise
    
    @property
    def firestore(self):
        """Get Firestore client with validation"""
        if not FirebaseManager._firestore_client:
            raise RuntimeError("Firestore client not initialized")
        return FirebaseManager._firestore_client
    
    @property
    def realtime_db(self):
        """Get Realtime Database reference"""
        if not FirebaseManager._realtime_db:
            raise RuntimeError("Realtime Database not initialized")
        return FirebaseManager._realtime_db
    
    def save_strategy_state(self, strategy_id: str, state: Dict[str, Any]) -> bool:
        """Save strategy state to Firestore"""
        try:
            doc_ref = self.firestore.collection('strategies').document(strategy_id)
            state['last_updated'] = datetime.utcnow().isoformat()
            state['version'] = state.get('version', 0) + 1
            doc_ref.set(state, merge=True)
            logger.debug(f"Strategy {strategy_id} state saved")
            return True
        except Exception as e:
            logger.error(f"Failed to save strategy state: {str(e)}")
            return False
    
    def get_strategy_state(self, strategy_id: str) -> Optional[Dict[str, Any]]:
        """Retrieve strategy state from Firestore"""
        try:
            doc_ref = self.firestore.collection('strategies').document(strategy_id)
            doc = doc_ref.get()
            return doc.to_dict() if doc.exists else None
        except Exception as e:
            logger.error(f"Failed to get strategy state: {str(e)}")
            return None
    
    def stream_metrics(self, metric_name: str, value: float, metadata: Dict[str, Any] = None) -> None:
        """Stream metrics to Realtime Database"""
        try:
            timestamp = datetime.utcnow().isoformat()
            metric_data = {
                'value': value,
                'timestamp