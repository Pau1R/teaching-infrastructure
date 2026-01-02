# Automated Testing System Architecture

## Overview
A GitHub-based automated testing system for Python students using discussions, workflows, and AI analysis.

## System Components

### 1. Teacher Trigger System Workflow (`trigger_test`)
- **Repository**: `python2026_tasks`
- **Trigger**: Manual workflow dispatch (`workflow_dispatch`)
- **Input**: Test number (e.g., "41.2")
- **Process**:
  1. Parse test number to extract lesson folder path
  2. Read student GitHub names from `name_mapping.md`
  3. For each student (100 ms delay between students):
     - Read test JSON from general file (`python2026_tasks/lessons/lesson {number}/tests/test {number}.json`) or student-specific file
     - Create discussion in their `python2026` repository
     - Discussion title: "Test {test_number} - {test_name}"
     - Add initial comment from `python2026_tasks/misc/inital_test_page.txt`
     - create file with test link in `python2026/lessons/lesson {number}/tests/test {number} {student_name}.md`

### 2. Test Watch System Workflow (`test_runner`)
- **Repository**: `python2026` (student repositories)
- **Trigger**: `discussion_comment` event
- **Implementation**: Each student's repository contains a starter workflow `test_runner.yml` that calls the centralized `test_runner` workflow from `python2026_tasks` repository
- **Verification Process**:
  1. Confirm first comment matches initial template content
  2. Verify comment author is the teacher (from `python2026_tasks`)
  3. Only proceed if both conditions are met

- **Initial Actions**:
  1. Load test JSON from `python2026_tasks/lessons/lesson {number}/tests/test {number}.json`
  2. Process questions:
     - Shuffle questions if `config.shuffle` is true
     - Always shuffle answer options (positions 1-5)
     - Keep "Я не знаю ответа" as the fixed 6th option
  3. Replace initial comment with first question

- **Polling Loop** (every 3 seconds):
  - State stored in workflow variables only
  - Fetch discussion comments via GitHub API
  - **Change Detection Logic**:
     - If answer detected (one or multiple task list checkbox 1-6 or new comment):
       - Save answer with scoring:
         - Correct answer: 1 point
         - Wrong answer: 0 points
         - "Я не знаю ответа": 0.2 points
       - Replace comment with next question
     - If unrelated content:
       - Delete excessive comments
       - Restore question from workflow variable

- **Test Completion Process**:
  1. Calculate final score and percentage
  2. Clear all discussion comments
  3. Wait for AI analysis to complete
  4. After results file is created, add new comment with:
     - Test completion message
     - Direct link to results file
  5. The comment is added only after the entire process is finished
  6. The percentage is appended to the start of the results file.
  7. The discussion is deleted
  8. The file with test link is deleted
  9. The workflow is completed

### 3. AI Analysis Integration
- **Prompt File**: Uses `python2026/misc/test_result_analysis_prompt.md`
- **Implementation**: Same approach as in `task_analysis.yml`
- **Process**:
  1. Collect all wrong answers
  2. Send to AI with context and prompt
  3. Receive detailed explanations
  4. Append analysis to results file
- **No Discussion Posting**: Results only saved to file

### 4. Results Storage
- **student location**: `/lessons/lesson {number}/tests/test_{test_number}_result.md`
- **teacher location**: `python2026_tasks/lessons/lesson {...}/tests/test {test_number} {student_name} result.md`
- **Content**:
  - test information
  - All questions and student answers
  - Wrong answers with explanations
  - Final score
  - AI analysis of wrong answers

### 5. Data Flow
- **Question Source**: `python2026_tasks/lessons/lesson {number}/tests/test {number}.json`
- **State Management**: Workflow variables only (no artifacts)
- **Student Mapping**: `python2026_tasks/name_mapping.md`
- **Initial Template**: `python2026_tasks/lessons/lesson {number}/tests/test_inital_example.txt`

### 6. Implementation Details
- **Answer Options**: Always 6 options (5 shuffled + fixed "Я не знаю ответа")
- **End test**: Checkbox for ending the test. Ask for confirmation
- **Scoring**: 1/0/0.2 point system
- **Comment Management**: Complete cleanup and single final comment
- **Security**: Teacher verification through comment author check
- **Rate Handling**: Polling every 3 seconds (1200 requests/hour, within safe limits)
- **Time Limit**: workflow hard limit 50 minutes

### 7. File Structure Example
```
python2026_tasks/
├── name_mapping.md
├── .github/workflows/
│   ├── trigger_test.yml  (creates discussions)
│   └── test_runner.yml  (main test logic)
└── lessons/lesson 41 (29.12). Сложность алгоритмов. Big O/
    └── tests/
        ├── test 41.2.json  (general test file)
        ├── test 41.2 EvgeniyN.md  (student test file)
        ├── test 41.2 EvgeniyN result.md  (student result file)
        └── test 41.2 results.csv  (list of results file)

python2026/students/{student_name}/
├── .github/workflows/
│   └── test_runner.yml  (starter workflow)
└── lessons/lesson 41 (29.12). Сложность алгоритмов. Big O/
    └── tests/
        ├── test 41.2 {student_name}.md  (link to discussion)
        └── test 41.2 {student_name} result.md
```

## Security Considerations
- Teacher verification through GitHub user check
- Test discussion validation via template matching
- Secure AI API key storage in repository secrets
- Access control through repository permissions
