# CodeGraph Improvements Plan

Based on code review, several features from the original Phase 2 plan have already been implemented. This document outlines the remaining work needed.

## Current Implementation Status

### ✅ Already Implemented
1. **Export Formats** (DOT, PlantUML) - Complete
   - `toDot()` method in GraphAnalyzer.ts
   - `toPlantUML()` method in GraphAnalyzer.ts
   - Format support in analyze_dependencies tool
   - Output schemas updated

2. **PageRank Metric** - Algorithm Complete, Dependency Missing
   - PageRank implementation in rankImpact() method
   - MCP tool schema updated
   - Missing: graphology-pagerank dependency

3. **Code Complexity Analysis** - Complete
   - ComplexityAnalyzer class implemented
   - analyze_complexity tool registered
   - All complexity metrics calculated

### ⏳ Remaining Work

## 0. Prerequisites (Immediate)
**Issue**: PageRank algorithm implemented but dependency not installed

**Actions**:
```bash
npm install graphology-pagerank
```

**Verification**:
- Check that PageRank metric works in rank_impact tool
- Ensure no runtime errors when using pagerank metric

## 1. Progress Indicators (High Priority)
**Issue**: No progress reporting for large codebase scans

**Location**: `src/parser/ProjectParser.ts`

**MCP Progress Reporting Clarification**:
- Progress is reported through tool response `structuredContent` fields
- No streaming or notifications - progress is synchronous with tool execution
- Clients poll for progress by re-invoking tools with progress callbacks
- Progress information is advisory and doesn't change tool behavior

**Actions**:
1. Add ProgressCallback type and ProgressInfo interface
2. Modify parse() method to accept onProgress option
3. Implement progress reporting during file parsing
4. Update MCP tools to support progress reporting via options
5. Add progress information to tool responses via structuredContent.progress

**Interface**:
```typescript
interface ProgressCallback {
  (progress: ProgressInfo): void;
}

interface ProgressInfo {
  phase: "scanning" | "parsing" | "analyzing";
  current: number;
  total: number;
  currentFile?: string;
  percentComplete: number;
}

// Extended tool input schemas to support progress
interface ToolOptions {
  onProgress?: ProgressCallback;
}
```

## 2. Error Message Improvements (High Priority)
**Issue**: Error messages lack context and suggestions

**Location**: `src/mcp/tools.ts`

**Actions**:
1. Audit all throw and catch blocks in tools.ts before implementing (approx 15 locations)
2. Enhance safeHandler to provide contextual error messages with error codes
3. Add specific error handling for common cases:
   - Directory not found → suggest checking path correctness and alternatives
   - No TypeScript files found → suggest checking extensions and ignore patterns
   - Too many files → suggest subdirectory scanning with examples
   - Permission denied → suggest checking file/directory permissions
   - Empty result → suggest directory may be empty or all files ignored
4. Return structured error codes (e.g., "ERR_DIRECTORY_NOT_FOUND") for better client handling

**Example Enhancement**:
```typescript
interface ErrorResponse {
  content: Array<{ type: "text"; text: string }>;
  structuredContent?: Record<string, unknown>;
  isError?: boolean;
  errorCode?: string; // e.g., "ERR_DIRECTORY_NOT_FOUND"
}

function getErrorMessage(error: Error, context: Record<string, unknown>): { message: string; errorCode: string } {
  const baseMessage = error.message;
  
  if (error.message.includes("does not exist")) {
    return {
      message: `${baseMessage}. Did you check the path is correct? Try using an absolute path.`,
      errorCode: "ERR_DIRECTORY_NOT_FOUND"
    };
  }
  
  if (error.message.includes("Too many files")) {
    return {
      message: `${baseMessage}. Try scanning a specific subdirectory like ./src or ./components`,
      errorCode: "ERR_TOO_MANY_FILES"
    };
  }
  
  if (error.message.includes("Path is not a directory")) {
    return {
      message: `${baseMessage}. Please provide a path to a directory, not a file.`,
      errorCode: "ERR_NOT_A_DIRECTORY"
    };
  }
  
  return {
    message: baseMessage,
    errorCode: "ERR_UNKNOWN"
  };
}
```

## 3. Minor Enhancements (Medium Priority)

### 3.1 Cache TTL Optimization
**Issue**: Analyzer cache never expires entries based on time

**Location**: `src/mcp/cache.ts`

**Risk Note**: Changes to eviction logic should be tested under load before merging as it could affect performance unpredictably.

**Actions**:
- Implement time-based eviction in addition to size-based eviction
- Add last accessed timestamp to cache entries
- Set default TTL to 30 minutes (configurable)
- Ensure thread-safe access to cache metadata

### 3.2 Search Enhancement
**Issue**: find_function tool could be enhanced with regex support

**Location**: `src/mcp/tools.ts`

**Actions**:
- Add optional regex parameter to find_function tool (default: false for backward compatibility)
- Implement case-insensitive regex matching when enabled
- Add validation to prevent ReDoS attacks (limit pattern complexity)
- Update tool documentation to explain regex usage

### 3.3 Documentation Improvements
**Issue**: Some internal methods lack JSDoc comments

**Location**: Throughout codebase

**Success Criterion**: JSDoc added to all public methods in GraphAnalyzer.ts, ComplexityAnalyzer.ts, and ProjectParser.ts; verified via typedoc --strict with zero warnings.

**Actions**:
- Add JSDoc comments to public methods
- Document complex algorithms (especially in GraphAnalyzer.ts)
- Add usage examples where appropriate
- Ensure all exported classes and interfaces have documentation

## Implementation Order

### Phase 0: Prerequisites (Immediate)
1. Install missing graphology-pagerank dependency
2. Verify PageRank functionality works correctly

### Phase 1: High Priority Features
1. Implement progress indicators in ProjectParser with MCP integration
2. Enhance error messages with contextual suggestions and error codes
3. Audit and improve all error handling in tools.ts

### Phase 2: Medium Priority Features
1. Implement cache TTL optimization (test under load before merging)
2. Add search enhancements (regex support with ReDoS protection)
3. Improve documentation and JSDoc coverage to meet success criteria

## Success Criteria

### Phase 0: Prerequisites
- [ ] graphology-pagerank installed successfully
- [ ] rank_impact tool works with "pagerank" metric
- [ ] No runtime errors when using PageRank

### Phase 1: High Priority
- [ ] Progress reporting functional for scan_codebase tool (via structuredContent.progress)
- [ ] Error messages include helpful suggestions and error codes
- [ ] All error handling in tools.ts audited and improved
- [ ] All existing tests continue to pass
- [ ] MCP tool backward compatibility maintained

### Phase 2: Medium Priority
- [ ] Cache implements both size and time-based eviction (tested under load)
- [ ] find_function supports optional regex search (with ReDoS protection)
- [ ] JSDoc added to all public methods in GraphAnalyzer.ts, ComplexityAnalyzer.ts, and ProjectParser.ts
- [ ] Type checking passes with no new warnings
- [ ] Zero warnings from typedoc --strict on targeted files

## Backward Compatibility
All changes must maintain backward compatibility:
- Existing MCP tool names and core functionality unchanged
- Existing output formats preserved (new fields optional)
- No breaking changes to public interfaces
- Progress reporting and regex features are opt-in
- Error codes are additive (don't change existing error structure)