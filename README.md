# AWS-Hackathon# AWS Hackathon Codebase Guide

## Architecture Overview

This is a full-stack document classification system with three main components:

1. **Backend (Flask)**: `app.py` - REST API orchestrating the ML pipeline
2. **ML Pipeline**: `aws_sagemaker/` (classification) + `aws_textract_project/` (OCR/extraction)
3. **Frontend (React)**: `awsfrontend/src/` - Multi-language UI with camera/file upload

### Data Flow

File → S3 upload → Textract (OCR) → SageMaker endpoint (classification) → Bedrock/Polly (enrichment) → Frontend display

The system classifies tax forms (W2 vs non-W2) and extracts their fields using AWS services.

## Critical Files & Patterns

### Configuration & Constants
- **S3 bucket**: Hardcoded as `'w2-datasets'` across all files (centralize if moving)
- **Label mapping**: `label_mapping.json` - maps class indices {0: "w2", 1: "non-w2"}
- **Endpoint config**: `endpoint_name.txt` + `label_mapping.json` required at runtime (copied to root from `aws_sagemaker/`)
- **Allowed file types**: `{'png', 'jpg', 'jpeg', 'pdf'}` in `textract_for_sagemaker.py`

### Backend Routes
- `POST /upload-and-process` - Main pipeline entry: validates file → uploads to S3 → processes through Textract → calls SageMaker
- Uses `CORS` enabled (necessary for frontend communication)
- Temporary files cleaned with `tempfile` module; S3 input folder cleared before each processing

### SageMaker Integration
- Predictor initialized in `aws_sagemaker/predict.py` with JSON serialization
- Confidence threshold check at `0.5` in `predict_text()` function
- Training datasets: 13 tax form types in `train_model.py` (1040_1, sch_a, sch_b, etc.)
- Output stored to `output/result/result.json` in S3 with structure: `{"predicted_label": "...", "confidence": ...}`

### Textract Processing Variants
- **W2 forms**: Use key-value pair extraction (`textract_ocr_better.py` → `get_kv_map/relationship`)
- **Non-W2 forms**: Fall back to line-based OCR (`textract_ocr_line.py`)
- Router logic in `aws_textract_project/pipeline.py` - checks predicted form type first, then branches extraction method
- Output: JSON with text content for Bedrock processing

### Frontend Conventions
- Language selection stored in `localStorage.selectedLanguage` (default: 'en')
- Uses `i18next` for translations; locales at `public/locales/{en,es,zh}/translation.json`
- Two-page SPA: `HomePage` (upload/capture) → `ResultsPage` (results display)
- Webcam integration via `react-webcam`; audio encoding with `lamejs`
- Navigation via `react-router-dom` v6

## Development Workflows

### Running Locally
```bash
# Backend
pip install -r aws_sagemaker/requirements.txt
python app.py  # Listens on Flask default (5000)

# Frontend
cd awsfrontend
npm install
npm start  # Runs on http://localhost:3000
```

### Training SageMaker Models
```bash
python aws_sagemaker/train_model.py --bucket-name w2-datasets --prefix datasets
```
Expects training data in S3 at `s3://w2-datasets/datasets/{form_type}/`

### Model Deployment
```bash
python aws_sagemaker/deploy_model.py
```
Generates `endpoint_name.txt` and `label_mapping.json` (copy to root if using app.py locally)

## Project-Specific Patterns

### AWS Credential Management
- All AWS clients initialized without explicit credentials → relies on IAM role in deployment
- SageMaker execution role hardcoded: `arn:aws:iam::715841371006:role/SageMakerExecutionRole`
- Development likely uses AWS CLI credentials or EC2 instance role

### Error Handling
- No custom exception classes; logging via `logging` module (INFO level in predict.py)
- S3 operations wrapped in try-catch with app.logger calls
- File validation uses `werkzeug.utils.secure_filename` before S3 upload

### State Management
- Form type detection happens in pipeline.py by downloading SageMaker prediction result first
- No explicit session/request ID tracking; relies on single-file-at-a-time processing
- Temporary files cleaned up via `os.remove()` after reading

### ML Model Quirks
- Scikit-learn models (implied by `train_test_split` import; likely sklearn.naive_bayes or similar)
- No explicit feature engineering visible; assumes Textract text features sufficient
- Confidence thresholds and label mapping configured at inference time, not training time

## Common Tasks

**Adding a new form type to classify:**
1. Add dataset name to `dataset_names` list in `train_model.py`
2. Upload training data to S3 at `s3://w2-datasets/datasets/{form_type}/`
3. Retrain and redeploy SageMaker model
4. Update `label_mapping.json` with new class index
5. Pipeline logic in `pipeline.py` will branch on `get_kv_map` vs line OCR based on predicted label

**Changing language support:**
- Add locale JSON at `awsfrontend/public/locales/{lang_code}/translation.json`
- Update language selector in `HomePage.js` to include new option
- Video instructions: add `instructions_{lang_code}.mp4` to public folder

**Debugging ML predictions:**
- Check form type prediction in S3: `s3://w2-datasets/output/result/result.json`
- If confidence < 0.5, check Textract OCR output quality and training data distribution
- Enable verbose logging by changing `logging.basicConfig(level=logging.DEBUG)` in predict.py
