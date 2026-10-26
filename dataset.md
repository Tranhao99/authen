# Overview
AIAH là một dự án của samsung dùng để quản lý các asset của AI như dataset, model, agent, sw component, app, product

## Chi tiết từng asset

### Dataset
- General
  + Acquision Route (*): Self Produced Data | Open Data | Purchased Data
  + Dataset Name (*): max_length=128
  + Dataset Type (*): Dropdown: Image | Video | Speech | Text | 3D Model | Sensor | Others
  + Summary (*): max_length=10000
  + Dataset Tag (*): #abc, #xyz
  + Task: (Audio) Audio Classification | (Audio) Audio-to-Audio | (Audio) Text-to-Audio | (CV) Depth Estimation | Image Classfication
  + Version: text, max_lengh=256
  + Button to copy dataset info from existing datasets 
- Data Card
  - Asset Information
    + Description (*): max_length=10000
    + Data URL (*): max_length=512 | N/A (checkbox)
    + URL Of Raw Data: max_length=2048
    + Language (*): select from dropdown | Irrelevant (checkbox)
    + Nation (*): select from dropdown | Irrelevant (checkbox)
    + Total Instances (*): Size (number), Unit (Image, Video, Video Clip, Audio, Hour, Text, Corpus, Utterance, Word, Sentence, Model, Persion, Others | Unknown
    + Total Dataset Size: Size (number) | Unit (KB, MB, GB, TB, Unknown)
    + Dataset Collection Method (*): list of checkboxes: Human (Expert) | Human (Ordinary people/Part-timer) | Algorithm | Crawling | Crowdsourcing | Unknown | Synthetic | Samsung Service | Others (Need to input detail)
    + Data Collection Process: text, max_length=256, ex: 1. Participant recruitment, 2 photographing 3. Annotation
    + Data Contract Period: start date, end date | Permanent (checkbox)
    + Detail Field By Dataset Type: Upload file (up to 100MB). (0 Bytes / 100 MB)

  - Asset Relation
    + Table of related asset
  - Publication Infromation
    + Publisher: max_length=256, ex: SR Health Solution Lab
    + Intended Use Cases: max_legnth=256, ex: Camera-based vital signal monitoring
    + Company: search company by keyword then select from list
    + Reference Paper: max_length=10000, placeholder: Add paper information introducing dataset or phases related to usage restrictions
- Legal
  - License Information
    + License URL: max_length=512
  - Privacy Information
    + Privacy Information (*): Radio buttons: None | Existance | Unknown
- AI Safety information
  + AI Safeguard Applicability/AI Safeguard: Radio buttons: Applicable | Not Applicable
- Review
  - Validation
    + Period (Production/ Service): start date, end date | Permanent (checkbox)
    + License Expiration Date: date | Permanent (checkbox)
    + Privacy Policy Notice (URL): max_length=512, ex: https://www.samsung.com
    + PI Rentention Period: start date, end date | Permanent (checkbox)
  - Review Result
    + Legal issue: max_length=10000, placeholder: Please enter whether any legal issues (Privacy info, Sensitive info, Confidential Info, Children info, etc) are involved and the results of the legal review.
- Sample
  + Upload file button: guide: You can upload multiple sample data (up to overall 100 MB) to help data consumers understand the dataset. (0 bytes / 100 MB)
