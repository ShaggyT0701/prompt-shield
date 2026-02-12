# Writing Custom Detectors

prompt-shield has a plugin architecture that makes it straightforward to add new detectors. This guide covers the interface, a complete template, and how to distribute your detector.

## The BaseDetector Interface

Every detector extends `BaseDetector` and implements one required method: `detect()`.

```python
from prompt_shield.detectors.base import BaseDetector
from prompt_shield.models import DetectionResult, MatchDetail, Severity

class BaseDetector(ABC):
    # Required class attributes
    detector_id: str       # Unique ID (convention: "d0XX_snake_case" or "custom_name")
    name: str              # Human-readable name
    description: str       # One-line description of what it detects
    severity: Severity     # LOW, MEDIUM, HIGH, or CRITICAL
    tags: list[str]        # Category tags (e.g., ["obfuscation", "custom"])
    version: str           # Semver version string
    author: str            # Author or organization name

    # Required method
    def detect(self, input_text: str, context: dict | None = None) -> DetectionResult: ...

    # Optional lifecycle hooks
    def setup(self, config: dict) -> None: ...    # Called once during engine init
    def teardown(self) -> None: ...               # Called on unregister
```

## Complete Template

```python
"""Detector template — copy this file and modify."""

import regex

from prompt_shield.detectors.base import BaseDetector
from prompt_shield.models import DetectionResult, MatchDetail, Severity


class MyCustomDetector(BaseDetector):
    """Detects [describe what this catches]."""

    detector_id: str = "custom_my_detector"
    name: str = "My Custom Detector"
    description: str = "Detects [specific pattern or technique]"
    severity: Severity = Severity.MEDIUM
    tags: list[str] = ["custom"]
    version: str = "1.0.0"
    author: str = "your-name"

    _patterns: list[tuple[str, str]] = [
        (r"pattern_one", "Description of what pattern_one matches"),
        (r"pattern_two", "Description of what pattern_two matches"),
    ]

    def setup(self, config: dict[str, object]) -> None:
        """Optional: read custom config values."""
        # Example: self._custom_setting = config.get("custom_setting", "default")
        pass

    def detect(
        self, input_text: str, context: dict[str, object] | None = None
    ) -> DetectionResult:
        matches: list[MatchDetail] = []

        for pattern_str, description in self._patterns:
            pattern = regex.compile(pattern_str, regex.IGNORECASE)
            for m in pattern.finditer(input_text):
                matches.append(
                    MatchDetail(
                        pattern=pattern_str,
                        matched_text=m.group(),
                        position=(m.start(), m.end()),
                        description=description,
                    )
                )

        if not matches:
            return DetectionResult(
                detector_id=self.detector_id,
                detected=False,
                confidence=0.0,
                severity=self.severity,
                explanation="No suspicious patterns found",
            )

        confidence = min(1.0, 0.8 + 0.1 * (len(matches) - 1))
        return DetectionResult(
            detector_id=self.detector_id,
            detected=True,
            confidence=confidence,
            severity=self.severity,
            matches=matches,
            explanation=f"Detected {len(matches)} pattern(s)",
        )
```

## Registration Methods

### Method 1: Runtime Registration

Register a detector instance directly with the engine:

```python
from prompt_shield import PromptShieldEngine

engine = PromptShieldEngine()
engine.register_detector(MyCustomDetector())
```

### Method 2: Entry Point (for packages)

Add an entry point to your `pyproject.toml`:

```toml
[project.entry-points."prompt_shield.detectors"]
my_detector = "my_package.detector:MyCustomDetector"
```

The engine auto-discovers entry point detectors on startup. No manual registration needed.

### Method 3: Auto-Discovery

If you are contributing directly to the prompt-shield codebase, place your detector file in `src/prompt_shield/detectors/` with the naming convention `dXXX_snake_case.py`. The registry auto-discovers all `BaseDetector` subclasses in that package.

## DetectionResult Fields

| Field | Type | Description |
|---|---|---|
| `detector_id` | `str` | Must match the detector's `detector_id` attribute |
| `detected` | `bool` | `True` if injection was found |
| `confidence` | `float` | 0.0 to 1.0. Must exceed the threshold to trigger an action |
| `severity` | `Severity` | LOW, MEDIUM, HIGH, or CRITICAL |
| `matches` | `list[MatchDetail]` | Specific patterns/positions matched |
| `explanation` | `str` | Human-readable explanation |
| `metadata` | `dict` | Arbitrary extra data |

## Guidelines

- Set `detected=False` and `confidence=0.0` when nothing is found
- Use `MatchDetail.position` (start, end) so that sanitizers can redact matched segments
- Keep confidence between 0.7 and 1.0 for genuine detections; the engine ignores results below the configured threshold (default 0.7)
- Use `regex` (not `re`) for consistency with the rest of the codebase
- Add clear `explanation` text; it appears in reports and logs
- Choose `severity` to reflect the real-world impact of the attack type

## Testing

Write tests in `tests/detectors/` using the standard pattern:

```python
def test_my_detector_detects_attack():
    det = MyCustomDetector()
    result = det.detect("input containing pattern_one")
    assert result.detected
    assert result.confidence >= 0.7

def test_my_detector_passes_safe_input():
    det = MyCustomDetector()
    result = det.detect("normal safe input")
    assert not result.detected
    assert result.confidence == 0.0
```
