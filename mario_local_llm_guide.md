# Mario PPO Local LLM Integration Guide

## Executive Summary

This guide provides a **minimal complexity** integration of local LLM capabilities into your existing Mario PPO Dashboard architecture. It follows your established patterns, quality standards, and coding conventions while adding AI-enhanced analysis with zero cloud dependencies.

## Integration Architecture

### Minimal Addition Strategy
```
Existing Mario PPO Dashboard
├── Core (config, exceptions, utils)
├── Game State Management  
├── Training Pipeline
├── API Layer
├── Event System
└── UI Components
    │
    └── NEW: Local LLM Integration Layer
        ├── llm_client.py (Ollama interface)
        ├── mario_ai_analyzer.py (AI analysis)
        └── llm_config.py (LLM configuration)
```

## Implementation Plan

### Phase 1: Core LLM Integration (1 hour)

#### 1. Add LLM Configuration
**File**: `src/core/llm_config.py`
```python
"""LLM Configuration Management following core config patterns."""

from typing import Dict, Any, Optional
from src.core.config import ConfigManager
from src.core.exceptions import ConfigError
import logging

logger = logging.getLogger(__name__)

class LLMConfig:
    """Local LLM configuration management."""
    
    DEFAULT_CONFIG = {
        'enabled': True,
        'provider': 'ollama',
        'base_url': 'http://localhost:11434',
        'model': 'llama3.1:8b',
        'timeout': 30,
        'max_tokens': 1000,
        'temperature': 0.7,
        'analysis_frequency': 100,  # Analyze every 100 episodes
        'features': {
            'training_analysis': True,
            'performance_insights': True,
            'hyperparameter_suggestions': True,
            'strategy_recommendations': False  # Disabled by default
        }
    }
    
    def __init__(self, config_manager: ConfigManager):
        """Initialize LLM configuration.
        
        Args:
            config_manager: Main configuration manager instance
        """
        self.config_manager = config_manager
        self._load_config()
    
    def _load_config(self) -> None:
        """Load LLM configuration with defaults."""
        try:
            # Get LLM section from main config
            llm_config = self.config_manager.get_section('llm')
            
            # Merge with defaults
            self.config = {**self.DEFAULT_CONFIG}
            if llm_config:
                self.config.update(llm_config)
                
            logger.info(f"LLM configuration loaded: provider={self.config['provider']}, "
                       f"model={self.config['model']}, enabled={self.config['enabled']}")
                       
        except Exception as e:
            logger.warning(f"Failed to load LLM config, using defaults: {e}")
            self.config = self.DEFAULT_CONFIG.copy()
    
    def get(self, key: str, default: Any = None) -> Any:
        """Get configuration value."""
        keys = key.split('.')
        value = self.config
        
        try:
            for k in keys:
                value = value[k]
            return value
        except (KeyError, TypeError):
            return default
    
    def is_enabled(self) -> bool:
        """Check if LLM integration is enabled."""
        return self.config.get('enabled', False)
    
    def should_analyze(self, episode: int) -> bool:
        """Check if analysis should be performed for this episode."""
        if not self.is_enabled():
            return False
        
        frequency = self.config.get('analysis_frequency', 100)
        return episode % frequency == 0
```

#### 2. Create LLM Client
**File**: `src/integration/llm_client.py`
```python
"""Simple Ollama client following integration patterns."""

import requests
import json
import logging
from typing import Dict, Any, Optional
from src.core.exceptions import MarioBaseException
from src.core.llm_config import LLMConfig

logger = logging.getLogger(__name__)

class LLMError(MarioBaseException):
    """LLM-specific error."""
    pass

class OllamaClient:
    """Simple Ollama client for local LLM interaction."""
    
    def __init__(self, config: LLMConfig):
        """Initialize Ollama client.
        
        Args:
            config: LLM configuration instance
        """
        self.config = config
        self.base_url = config.get('base_url', 'http://localhost:11434')
        self.model = config.get('model', 'llama3.1:8b')
        self.timeout = config.get('timeout', 30)
        self.session = requests.Session()
        
    def is_available(self) -> bool:
        """Check if Ollama is available."""
        try:
            response = self.session.get(
                f"{self.base_url}/api/tags",
                timeout=5
            )
            return response.status_code == 200
        except Exception as e:
            logger.debug(f"Ollama not available: {e}")
            return False
    
    def generate(self, prompt: str, **kwargs) -> Optional[str]:
        """Generate text using Ollama.
        
        Args:
            prompt: Input prompt
            **kwargs: Additional generation parameters
            
        Returns:
            Generated text or None if failed
        """
        if not self.config.is_enabled():
            return None
            
        try:
            payload = {
                'model': self.model,
                'prompt': prompt,
                'stream': False,
                'options': {
                    'temperature': kwargs.get('temperature', self.config.get('temperature', 0.7)),
                    'num_predict': kwargs.get('max_tokens', self.config.get('max_tokens', 1000))
                }
            }
            
            response = self.session.post(
                f"{self.base_url}/api/generate",
                json=payload,
                timeout=self.timeout
            )
            
            if response.status_code == 200:
                result = response.json()
                return result.get('response', '').strip()
            else:
                logger.warning(f"Ollama request failed: {response.status_code}")
                return None
                
        except Exception as e:
            logger.error(f"LLM generation failed: {e}")
            return None
```

#### 3. Create Mario AI Analyzer
**File**: `src/training/mario_ai_analyzer.py`
```python
"""Mario-specific AI analysis following training module patterns."""

import logging
from typing import Dict, Any, Optional
from src.integration.llm_client import OllamaClient
from src.core.llm_config import LLMConfig
from src.events.event_system import EventSystem

logger = logging.getLogger(__name__)

class MarioAIAnalyzer:
    """AI-powered analysis for Mario PPO training."""
    
    def __init__(self, config: LLMConfig, event_system: EventSystem):
        """Initialize Mario AI analyzer.
        
        Args:
            config: LLM configuration
            event_system: Event system for notifications
        """
        self.config = config
        self.event_system = event_system
        self.llm_client = OllamaClient(config)
        self.analysis_count = 0
        
        # Check availability on startup
        if config.is_enabled():
            available = self.llm_client.is_available()
            logger.info(f"Mario AI Analyzer initialized: LLM available={available}")
            if not available:
                logger.warning("LLM not available - analysis will be skipped")
    
    def analyze_training_episode(self, episode_data: Dict[str, Any]) -> Optional[str]:
        """Analyze a training episode.
        
        Args:
            episode_data: Episode statistics and metrics
            
        Returns:
            Analysis text or None if analysis not performed
        """
        if not self.config.is_enabled():
            return None
            
        if not self.config.should_analyze(episode_data.get('episode', 0)):
            return None
            
        if not self.llm_client.is_available():
            return None
        
        try:
            # Create analysis prompt
            prompt = self._create_episode_analysis_prompt(episode_data)
            
            # Generate analysis
            analysis = self.llm_client.generate(prompt)
            
            if analysis:
                self.analysis_count += 1
                
                # Emit analysis event
                self.event_system.emit_event({
                    'event_type': 'ai_analysis_complete',
                    'data': {
                        'episode': episode_data.get('episode'),
                        'analysis': analysis,
                        'analysis_count': self.analysis_count
                    }
                })
                
                logger.info(f"AI analysis completed for episode {episode_data.get('episode')}")
                
            return analysis
            
        except Exception as e:
            logger.error(f"Episode analysis failed: {e}")
            return None
    
    def suggest_hyperparameters(self, training_metrics: Dict[str, Any]) -> Optional[str]:
        """Suggest hyperparameter adjustments.
        
        Args:
            training_metrics: Current training performance metrics
            
        Returns:
            Hyperparameter suggestions or None
        """
        if not self.config.get('features.hyperparameter_suggestions', True):
            return None
            
        if not self.llm_client.is_available():
            return None
        
        try:
            prompt = self._create_hyperparameter_prompt(training_metrics)
            suggestions = self.llm_client.generate(prompt)
            
            if suggestions:
                logger.info("AI hyperparameter suggestions generated")
                
            return suggestions
            
        except Exception as e:
            logger.error(f"Hyperparameter suggestion failed: {e}")
            return None
    
    def _create_episode_analysis_prompt(self, episode_data: Dict[str, Any]) -> str:
        """Create prompt for episode analysis."""
        return f"""Analyze this Super Mario Bros PPO training episode:

Episode: {episode_data.get('episode', 'Unknown')}
World-Stage: {episode_data.get('world', '?')}-{episode_data.get('stage', '?')}
Total Reward: {episode_data.get('total_reward', 0)}
Episode Length: {episode_data.get('episode_length', 0)} steps
Deaths: {episode_data.get('deaths', 0)}
Score: {episode_data.get('score', 0)}
Completion: {'Yes' if episode_data.get('completed', False) else 'No'}

Recent Performance:
- Average Reward (last 10): {episode_data.get('avg_reward_10', 0):.2f}
- Success Rate: {episode_data.get('success_rate', 0):.2%}

Provide a brief analysis (2-3 sentences) focusing on:
1. Performance assessment
2. One specific improvement suggestion

Keep response under 150 words."""

    def _create_hyperparameter_prompt(self, metrics: Dict[str, Any]) -> str:
        """Create prompt for hyperparameter suggestions."""
        return f"""Analyze these Mario PPO training metrics and suggest ONE hyperparameter adjustment:

Current Performance:
- Average Reward: {metrics.get('avg_reward', 0):.2f}
- Policy Loss: {metrics.get('policy_loss', 0):.6f}
- Value Loss: {metrics.get('value_loss', 0):.6f}
- Learning Rate: {metrics.get('learning_rate', 0):.6f}
- Episodes Trained: {metrics.get('episodes', 0)}

Current Settings:
- Learning Rate: {metrics.get('current_lr', 3e-4):.6f}
- Gamma: {metrics.get('gamma', 0.99)}
- GAE Lambda: {metrics.get('gae_lambda', 0.95)}

Suggest ONE specific hyperparameter change with brief reasoning (under 100 words)."""

    def get_status(self) -> Dict[str, Any]:
        """Get analyzer status information."""
        return {
            'enabled': self.config.is_enabled(),
            'llm_available': self.llm_client.is_available() if self.config.is_enabled() else False,
            'model': self.config.get('model'),
            'analysis_count': self.analysis_count,
            'features': self.config.get('features', {})
        }
```

### Phase 2: Integration with Training Loop (30 minutes)

#### 4. Modify Training Loop
**File**: `src/training/training_loop.py` (Add these methods)
```python
# Add to existing TrainingLoop class imports:
from src.training.mario_ai_analyzer import MarioAIAnalyzer
from src.core.llm_config import LLMConfig

# Add to TrainingLoop.__init__:
def __init__(self, agent: PPOAgent, environment_manager: EnvironmentManager, 
             config: Dict[str, Any]):
    # ... existing initialization ...
    
    # Initialize AI analyzer
    llm_config = LLMConfig(self.config_manager)
    self.ai_analyzer = MarioAIAnalyzer(llm_config, self.event_system)

# Add new method to TrainingLoop:
def _process_episode_completion(self, episode_data: Dict[str, Any]) -> None:
    """Process episode completion with AI analysis."""
    # Existing episode processing...
    
    # AI Analysis (non-blocking)
    try:
        analysis = self.ai_analyzer.analyze_training_episode(episode_data)
        if analysis:
            episode_data['ai_analysis'] = analysis
            logger.info(f"Episode {episode_data['episode']}: {analysis}")
    except Exception as e:
        logger.debug(f"AI analysis skipped: {e}")
    
    # Continue with existing logic...
```

### Phase 3: Configuration & Setup (15 minutes)

#### 5. Add LLM Configuration Section
**File**: Update your main configuration file (e.g., `config/config.yaml`)
```yaml
# Add this section to your existing config
llm:
  enabled: true
  provider: "ollama"
  base_url: "http://localhost:11434"
  model: "llama3.1:8b"
  timeout: 30
  max_tokens: 1000
  temperature: 0.7
  analysis_frequency: 100
  features:
    training_analysis: true
    performance_insights: true
    hyperparameter_suggestions: true
    strategy_recommendations: false
```

#### 6. Simple Setup Script
**File**: `scripts/setup_local_llm.sh`
```bash
#!/bin/bash
# Simple Ollama setup for Mario PPO

echo "Setting up Local LLM for Mario PPO..."

# Install Ollama (Linux/macOS)
if ! command -v ollama &> /dev/null; then
    echo "Installing Ollama..."
    curl -fsSL https://ollama.com/install.sh | sh
else
    echo "Ollama already installed"
fi

# Start Ollama service
echo "Starting Ollama service..."
ollama serve &
sleep 5

# Pull the model
echo "Downloading model (this may take a few minutes)..."
ollama pull llama3.1:8b

# Test the setup
echo "Testing LLM setup..."
response=$(ollama generate llama3.1:8b "Hello! Can you analyze Mario gameplay?" --timeout 10)

if [ $? -eq 0 ]; then
    echo "✅ Local LLM setup complete!"
    echo "Your Mario PPO dashboard now has AI analysis capabilities."
else
    echo "❌ Setup failed. Check Ollama installation."
fi
```

### Phase 4: Optional Dashboard Integration (15 minutes)

#### 7. Add AI Analysis to Dashboard
**File**: `src/ui/dashboard_components.py` (Add this component)
```python
# Add AI Analysis section to your existing dashboard
def render_ai_analysis_section(self, episode_data: Dict[str, Any]) -> str:
    """Render AI analysis section."""
    if not episode_data.get('ai_analysis'):
        return ""
    
    return f"""
    <div class="ai-analysis-section">
        <h3>🤖 AI Analysis</h3>
        <div class="ai-analysis-content">
            {episode_data['ai_analysis']}
        </div>
        <small>Generated by local LLM</small>
    </div>
    """
```

## Strict AI Instructions for Implementation

### Code Quality Requirements
```yaml
MANDATORY_REQUIREMENTS:
  - Follow existing code patterns EXACTLY
  - Use established error handling (MarioBaseException hierarchy)
  - Implement comprehensive logging using existing logger patterns
  - Add proper type hints following project conventions
  - Write docstrings in Google style matching existing code
  - Create unit tests with >95% coverage following test_framework.py
  - Use existing configuration management patterns
  - Follow event system patterns for notifications
  - No package.json modifications (Python project)
  - No duplicate code - reuse existing utilities

FORBIDDEN_ACTIONS:
  - Do NOT modify existing files without explicit instruction
  - Do NOT add new external dependencies without approval
  - Do NOT create duplicate configuration systems
  - Do NOT bypass existing error handling
  - Do NOT ignore the event system
  - Do NOT add complex dependencies
  - Do NOT create package.json (this is Python)
  - Do NOT overwrite existing files (Zero Overwrites principle)

TESTING_REQUIREMENTS:
  - Create tests in tests/unit/integration/
  - Follow TestFramework patterns exactly
  - Use MockEnvironment and MockAgent for testing
  - Achieve >95% line coverage, >90% branch coverage
  - Test error conditions and edge cases
  - Test configuration validation
  - Test LLM unavailability scenarios
```

### File Creation Checklist
```markdown
BEFORE CREATING ANY FILE:
- [ ] Check if similar functionality exists
- [ ] Follow existing module structure
- [ ] Use established import patterns
- [ ] Match existing naming conventions
- [ ] Implement proper error handling
- [ ] Add comprehensive logging
- [ ] Include type annotations
- [ ] Write complete docstrings
- [ ] Plan corresponding tests

AFTER CREATING FILES:
- [ ] Create unit tests
- [ ] Verify integration points
- [ ] Test error scenarios
- [ ] Validate configuration
- [ ] Check logging output
- [ ] Verify event emissions
- [ ] Test with LLM unavailable
- [ ] Update documentation
```

## Quality Assurance Validation

### Testing Strategy
1. **Unit Tests**: Test each component in isolation using mocks
2. **Integration Tests**: Test LLM integration with training loop
3. **Configuration Tests**: Test all configuration scenarios
4. **Error Handling Tests**: Test failure modes and graceful degradation
5. **Performance Tests**: Ensure no impact on training performance

### Success Criteria
- ✅ Zero modification to existing core files
- ✅ Graceful degradation when LLM unavailable  
- ✅ No impact on training performance
- ✅ Follows all existing patterns and conventions
- ✅ >95% test coverage
- ✅ Comprehensive error handling
- ✅ Complete integration with event system

## Usage Examples

### Basic Usage
```python
# The integration works automatically once configured
# Training loop will now include AI analysis every 100 episodes

# Check AI analyzer status
ai_status = training_loop.ai_analyzer.get_status()
print(f"AI Analysis: {'Enabled' if ai_status['enabled'] else 'Disabled'}")
```

### Manual Analysis
```python
# Force analysis for specific episode
episode_data = {
    'episode': 1000,
    'total_reward': 325.5,
    'world': 1,
    'stage': 1,
    'completed': True,
    'deaths': 2
}

analysis = ai_analyzer.analyze_training_episode(episode_data)
if analysis:
    print(f"AI Analysis: {analysis}")
```

## Troubleshooting

### Common Issues
1. **LLM Not Available**: Analysis is automatically skipped with debug logging
2. **Model Not Found**: Check Ollama model installation with `ollama list`
3. **Slow Responses**: Increase timeout in configuration
4. **Memory Issues**: Use smaller model like `llama3.1:8b` instead of larger variants

### Verification Commands
```bash
# Check Ollama status
curl http://localhost:11434/api/tags

# Test model directly
ollama run llama3.1:8b "Test message"

# Check logs
grep -i "llm\|ai.*analy" logs/mario_ppo.log
```

This integration provides AI-enhanced analysis while maintaining your project's sophisticated architecture and quality standards. The implementation is minimal, follows your established patterns, and gracefully handles all failure modes.