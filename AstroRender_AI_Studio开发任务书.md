# AstroRender 后端 - AI Studio 开发任务书

## 项目概况

- **技术栈**：Vercel Serverless Functions (Node.js/TypeScript), Hono, @vercel/kv, @vercel/blob, jose, @google/generative-ai, sharp [v2.2-added]
- **核心流程**：用户上传天文照片 → Astrometry.net plate-solving → Gemini 2.5 Flash Image 增强 → 二次验证 → 返回增强图片
- **每次请求生成3张增强版本供选择**

## 环境变量清单

| 变量名 | 说明 | 必填 | 示例 |
|--------|------|------|------|
| `ASTROMETRY_API_KEY` | Astrometry.net API Key | 是 | `your_astrometry_api_key` |
| `GEMINI_API_KEY` | Gemini API Key | 是 | `your_gemini_api_key` |
| `JWT_SECRET` | JWT签名密钥（至少32字符） | 是 | `your-32-character-secret-key-here` |
| `KV_REST_API_URL` | Vercel KV API URL | 自动 | Vercel自动注入 |
| `KV_REST_API_TOKEN` | Vercel KV Token | 自动 | Vercel自动注入 |
| `BLOB_READ_WRITE_TOKEN` | Vercel Blob Token | 自动 | Vercel自动注入 |
| `MAX_FILE_SIZE` | 最大文件大小（字节） | 否 | `20971520` (20MB) |
| `ASTROMETRY_TIMEOUT` | Astrometry超时（秒） | 否 | `300` (5分钟) |
| `MAX_FREE_QUOTA_PER_USER` | Free用户每日限额 | 否 | `3` |
| `MAX_PRO_QUOTA_PER_USER` | Pro用户每日限额 | 否 | `30` |
| `GEMINI_BATCH_SIZE` | Cron并行处理批次大小 | 否 | `5` |
| `CONSTELLATION_MATCH_THRESHOLD` | 星座匹配阈值 [v2.2-added] | 否 | `0.6` |

---

## Task 1: 类型定义与常量 — lib/types.ts + lib/constants.ts

### 目标
定义项目中使用的所有TypeScript接口和常量，为其他模块提供类型支持。

### 依赖
- **引入包**：无
- **引入模块**：无
- **使用类型**：本Task定义，供其他Task使用

### 类型定义（lib/types.ts）

```typescript
// User types
export type UserPlan = "free" | "pro" | "enterprise";

export interface JWTPayload {
  user_id: string;
  email?: string;
  plan: UserPlan;
  iat: number;
  exp: number;
}

export interface User {
  user_id: string;
  email: string;
  password_hash: string;
  plan: UserPlan;
  created_at: number;
}

export interface Session {
  user_id: string;
  plan: UserPlan;
  created_at: number;
}

// Task types
export type TaskStatus =
  | "uploaded"
  | "plate_solving"
  | "plate_solved"
  | "plate_failed"
  | "enhancing"
  | "verifying"
  | "completed"
  | "verification_failed"
  | "overlaying"           // [v2.2-added]
  | "overlay_verifying"    // [v2.2-revised] 叠加图二次验证中
  | "overlay_passed"       // [v2.2-revised] 叠加验证通过（临时状态）
  | "overlay_failed";      // [v2.2-added]

export interface AstrometryData {
  ra: string;           // e.g., "05h 35m 27.8s"
  dec: string;          // e.g., "+22° 02' 59.1\""
  objects: string[];    // e.g., ["M 1", "NGC 1952"]
  pixscale: number;     // arcsec/pixel
  radius: number;       // degrees
  orientation: number;  // degrees
}

export interface Task {
  task_id: string;
  status: TaskStatus;
  userId: string;
  subid?: string;        // Astrometry submission ID
  jobId?: string;       // Astrometry job ID
  originalUrl?: string;
  enhancedUrl?: string;
  astrometry?: AstrometryData;
  style?: "natural" | "vivid";
  verification?: VerificationResult;
  detectedConstellations?: DetectedConstellation[];  // [v2.2-added] plate-solving后填充
  constellation?: string;            // [v2.2-added] 当前叠加星座
  constellationStyle?: ConstellationStyle;  // [v2.2-added] 当前叠加风格
  constellationOverlayUrl?: string;  // [v2.2-added] 星座叠加结果URL
  constellationOverlayVerification?: OverlayVerificationResult;  // [v2.2-revised] 星座叠加验证结果
  error?: string;
  createdAt: number;
  updatedAt?: number;
}

export interface VerificationResult {
  authentic: boolean;
  checks?: {
    star_count: { pass: boolean; detail?: string };
    star_position: { pass: boolean; detail?: string };
    no_fabricated_objects: { pass: boolean; detail?: string };
    no_artifacts: { pass: boolean; detail?: string };
    color_authenticity: { pass: boolean; detail?: string };
    dynamic_range: { pass: boolean; detail?: string };
  };
  issues?: string[];
  verdict?: "AUTHENTIC" | "QUESTIONABLE" | "FABRICATED";
  skipped?: boolean;
}

// Constellation overlay verification result [v2.2-revised]
export interface OverlayVerificationResult {
  authentic: boolean;
  checks: {
    constellation_match: { pass: boolean; detail: string };
    star_alignment: { pass: boolean; detail: string };
    no_fabricated_objects: { pass: boolean; detail: string };
    style_consistency: { pass: boolean; detail: string };
  };
  issues?: string[];
  verdict: "PASS" | "FAIL";
}

// Quota types
export interface QuotaInfo {
  user_id: string;
  plan: UserPlan;
  daily_limit: number;
  used_today: number;
  remaining: number;
  resets_at: string;
}

// API Response types
export interface ApiResponse<T = unknown> {
  success?: boolean;
  task_id?: string;
  status?: TaskStatus;
  error?: string;
  user_message?: string;
  code?: number;
  data?: T;
}

export interface UploadResponse extends ApiResponse {
  task_id: string;
  status: "uploaded";
  message: string;
  estimated_wait_seconds: number;
}

export interface StatusResponse extends ApiResponse {
  task_id: string;
  status: TaskStatus;
  progress: {
    step: string;
    detail: string;
  };
  estimated_wait_seconds: number;
  astrometry?: AstrometryData;
}

export interface ResultResponse extends ApiResponse {
  task_id: string;
  status: TaskStatus;
  original_url?: string;
  enhanced_url?: string;
  annotated_info?: {
    objects: string[];
    constellation?: string;
    coordinates: string;
  };
  verification?: VerificationResult;
  detected_constellations?: DetectedConstellation[];    // [v2.2-added]
  constellation_overlay_url?: string;                    // [v2.2-added]
  constellation_overlay_info?: {                        // [v2.2-added]
    constellation: string;
    style: ConstellationStyle;
  };
  constellation_overlay_verification?: OverlayVerificationResult;  // [v2.2-revised] 星座叠加验证结果
}

// Constellation overlay types [v2.2-added] [v2.2-revised]
export type ConstellationStyle = "vintage";

export interface ConstellationStar {
  name: string;        // "Betelgeuse"
  ra: number;          // 赤经(度)
  dec: number;         // 赤纬(度)
  magnitude: number;   // 视星等
  is_primary: boolean; // 是否主星(用于连线)
}

export interface ConstellationDef {
  name: string;                      // "orion"
  display_name_en: string;           // "Orion"
  display_name_zh: string;           // "猎户座"
  scientific_name: string;           // "Orion" (天文学名)
  stars: ConstellationStar[];
  lines: [number, number][];         // 连线索引对
  mythology_description: string;     // 神话形象英文描述
  title: string;                     // "猎户座\nORION" [v2.2-revised]
  bounds: {
    ra_min: number;
    ra_max: number;
    dec_min: number;
    dec_max: number;
  };
  primary_star_count: number;
}

export interface DetectedConstellation {
  name: string;
  display_name: string;
  star_match_ratio: number;
  // [v2.2-revised] available_styles 已移除，风格已固定为 vintage
}

export interface StyleConfig {
  border_color: string;
  border_width: number;
  grid_color: string;
  grid_width: number;
  line_color: string;
  line_width: number;
  line_opacity: number;
  star_radius: number;
  star_color: string;
  title_color: string;
}
```

### 常量定义（lib/constants.ts）

```typescript
// Index collection keys
export const PENDING_PLATE_TASKS = "pending_plate_tasks";
export const PENDING_ENHANCE_TASKS = "pending_enhance_tasks";
export const PENDING_VERIFY_TASKS = "pending_verify_tasks";

// KV key prefixes
export const USER_PREFIX = "user_";
export const SESSION_PREFIX = "session:";
export const QUOTA_PREFIX = "quota_";
export const TASK_PREFIX = "task:";

// Cron lock keys
export const CRON_LOCK_POLL_ASTROMETRY = "cron:poll-astrometry:lock";
export const CRON_LOCK_PROCESS_ENHANCE = "cron:process-enhance:lock";

// Quota limits
export const FREE_DAILY_LIMIT = 3;
export const PRO_DAILY_LIMIT = 30;
export const ENTERPRISE_DAILY_LIMIT = Infinity;

// File constraints
export const MAX_FILE_SIZE = 20 * 1024 * 1024; // 20MB
export const ALLOWED_EXTENSIONS = [".jpg", ".jpeg", ".png", ".fits", ".fit"];
export const ALLOWED_MIME_TYPES = [
  "image/jpeg",
  "image/png",
  "application/fits",
  "application/x-fits",
  "application/fits-image",
];

// Astrometry settings
export const ASTROMETRY_BASE_URL = "http://nova.astrometry.net/api";
export const ASTROMETRY_TIMEOUT = 300; // 5 minutes
export const ASTROMETRY_UPLOAD_TIMEOUT = 30; // 30 seconds
export const ASTROMETRY_POLL_INTERVAL = 5; // 5 seconds

// Gemini settings
export const GEMINI_MODEL_ENHANCE = "gemini-2.5-flash-preview-04-17";
export const GEMINI_MODEL_VERIFY = "gemini-2.5-flash-preview-04-17";
export const GEMINI_ENHANCE_TIMEOUT = 60; // 60 seconds
export const GEMINI_VERIFY_TIMEOUT = 30; // 30 seconds

// Processing
export const DEFAULT_STYLE = "natural";
export const DEFAULT_DENOISE_LEVEL = 3;
export const DEFAULT_STRETCH_INTENSITY = 3;
export const ENHANCED_VERSION_COUNT = 3; // Generate 3 versions

// JWT settings
export const JWT_ISSUER = "astrorender-backend";
export const JWT_EXPIRES_IN = "7d";

// Cron batch sizes
export const PLATE_POLL_BATCH_SIZE = 10;
export const ENHANCE_POLL_BATCH_SIZE = 5;
export const VERIFY_POLL_BATCH_SIZE = 10;

// Timeout protection
export const SSCAN_TIMEOUT_MS = 5000;
export const SSCAN_BATCH_SIZE = 100;

// Constellation overlay [v2.2-added] [v2.2-revised]
export const CONSTELLATION_MATCH_THRESHOLD = 0.6;
export const PENDING_OVERLAY_TASKS = "pending_overlay_tasks";
export const PENDING_OVERLAY_VERIFY_TASKS = "pending_overlay_verify_tasks";  // [v2.2-revised] 叠加验证待处理队列

// [v2.2-revised] 风格已精简为单一vintage风格
export const CONSTELLATION_STYLES: Record<ConstellationStyle, StyleConfig> = {
  vintage: {
    border_color: "#C5A55A",
    border_width: 4,
    grid_color: "#8B7355",
    grid_width: 0.3,
    line_color: "#FFFFFF",
    line_width: 1,
    line_opacity: 0.6,
    star_radius: 3,
    star_color: "#FFFFFF",
    title_color: "#C5A55A",
  },
};
```

---

## Task 2: JWT认证工具 — lib/auth.ts + middleware/auth.ts

### 目标
实现JWT令牌的生成、验证和用户认证中间件。

### 依赖
- **引入包**：`jose` (JWT处理)
- **引入模块**：`lib/types.ts`, `lib/constants.ts`
- **使用类型**：`JWTPayload`, `UserPlan`

### 函数签名

```typescript
// lib/auth.ts
import { SignJWT, jwtVerify, decodeJwt, JWTPayload as JosePayload } from "jose";
import { JWT_SECRET, JWT_ISSUER, JWT_EXPIRES_IN } from "./constants";
import type { JWTPayload, UserPlan } from "./types";

// Verify JWT_SECRET exists at module load time [P0-C]
const JWT_SECRET_KEY = new TextEncoder().encode(JWT_SECRET);

/**
 * Generate JWT token for a user
 */
export async function generateToken(
  userId: string,
  plan: UserPlan,
  email?: string
): Promise<string> {
  const payload: Omit<JWTPayload, "iat" | "exp"> = {
    user_id: userId,
    plan,
    ...(email && { email }),
  };

  return new SignJWT(payload as unknown as JosePayload)
    .setProtectedHeader({ alg: "HS256" })
    .setIssuedAt()
    .setIssuer(JWT_ISSUER)
    .setExpirationTime(JWT_EXPIRES_IN)
    .sign(JWT_SECRET_KEY);
}

/**
 * Verify JWT token and return payload
 */
export async function verifyToken(token: string): Promise<JWTPayload | null> {
  try {
    const { payload } = await jwtVerify(token, JWT_SECRET_KEY);
    return payload as unknown as JWTPayload;
  } catch {
    return null;
  }
}

/**
 * Decode JWT without verification (for extracting user_id only)
 */
export function decodeToken(token: string): JWTPayload | null {
  try {
    const payload = decodeJwt(token);
    return payload as unknown as JWTPayload;
  } catch {
    return null;
  }
}
```

```typescript
// middleware/auth.ts
import { verifyToken, decodeToken } from "../lib/auth";
import type { JWTPayload } from "../lib/types";

/**
 * Authentication middleware - verify JWT in Authorization header
 * Returns Response with error if unauthorized, null if authorized
 */
export async function authMiddleware(req: Request): Promise<Response | null> {
  const authHeader = req.headers.get("Authorization");

  if (!authHeader || !authHeader.startsWith("Bearer ")) {
    return Response.json(
      {
        error: "UNAUTHORIZED",
        user_message: "请先登录后使用",
        code: 401,
      },
      { status: 401 }
    );
  }

  const token = authHeader.slice(7);
  const payload = await verifyToken(token);

  if (!payload) {
    return Response.json(
      {
        error: "INVALID_TOKEN",
        user_message: "登录已过期，请重新登录",
        code: 401,
      },
      { status: 401 }
    );
  }

  return null; // Verification passed, continue
}

/**
 * Extract user_id from request Authorization header
 * Returns null if no valid token found
 */
export function getUserId(req: Request): string | null {
  const authHeader = req.headers.get("Authorization");
  if (!authHeader || !authHeader.startsWith("Bearer ")) return null;

  const payload = decodeToken(authHeader.slice(7));
  return payload?.user_id ?? null;
}

/**
 * Get full JWT payload from request
 */
export function getJWTPayload(req: Request): JWTPayload | null {
  const authHeader = req.headers.get("Authorization");
  if (!authHeader || !authHeader.startsWith("Bearer ")) return null;

  return decodeToken(authHeader.slice(7));
}
```

### 安全约束
- **JWT_SECRET无默认值**：环境变量不存在时在模块加载时直接抛错
- **配额从JWT移除**：daily_quota不在JWT中，实时查询KV保证一致性
- **使用jose库**：`decodeJwt`用于解析，`jwtVerify`用于验证

---

## Task 3: Session/Quota管理 — lib/session-store.ts

### 目标
管理用户Session和配额检查逻辑，使用Vercel KV存储。

### 依赖
- **引入包**：`@vercel/kv`
- **引入模块**：`lib/types.ts`, `lib/constants.ts`
- **使用类型**：`Session`, `UserPlan`, `QuotaInfo`

### 函数签名

```typescript
import { kv } from "@vercel/kv";
import { USER_PREFIX, SESSION_PREFIX, QUOTA_PREFIX, FREE_DAILY_LIMIT, PRO_DAILY_LIMIT } from "./constants";
import type { Session, UserPlan, QuotaInfo } from "./types";

/**
 * Create or update user session
 */
export async function createSession(
  userId: string,
  plan: UserPlan
): Promise<Session> {
  const sessionKey = `${USER_PREFIX}${SESSION_PREFIX}${userId}`;
  const session: Session = {
    user_id: userId,
    plan,
    created_at: Date.now(),
  };
  await kv.set(sessionKey, session, { ex: 604800 }); // 7 days expiry
  return session;
}

/**
 * Get user session
 */
export async function getSession(userId: string): Promise<Session | null> {
  const sessionKey = `${USER_PREFIX}${SESSION_PREFIX}${userId}`;
  return await kv.get<Session>(sessionKey);
}

/**
 * Get user's daily usage count
 */
export async function getDailyUsage(userId: string): Promise<number> {
  const today = new Date().toISOString().split("T")[0];
  const key = `${QUOTA_PREFIX}${userId}:${today}`;
  const usage = await kv.get<number>(key);
  return usage || 0;
}

/**
 * Increment user's daily usage
 */
export async function incrementDailyUsage(userId: string): Promise<number> {
  const today = new Date().toISOString().split("T")[0];
  const key = `${QUOTA_PREFIX}${userId}:${today}`;
  const newCount = await kv.incr(key);

  // Set expiry for next day midnight
  const tomorrow = new Date();
  tomorrow.setDate(tomorrow.getDate() + 1);
  tomorrow.setHours(0, 0, 0, 0);
  const ttl = Math.floor((tomorrow.getTime() - Date.now()) / 1000);
  await kv.expire(key, ttl);

  return newCount;
}

/**
 * Get daily limit based on plan
 */
function getDailyLimit(plan: UserPlan): number {
  switch (plan) {
    case "pro":
      return PRO_DAILY_LIMIT;
    case "enterprise":
      return Infinity;
    default:
      return FREE_DAILY_LIMIT;
  }
}

/**
 * Check if user has remaining quota
 */
export async function checkQuota(userId: string): Promise<{
  allowed: boolean;
  remaining: number;
  limit: number;
  used: number;
}> {
  const session = await getSession(userId);
  const plan = session?.plan || "free";
  const limit = getDailyLimit(plan);
  const used = await getDailyUsage(userId);
  const remaining = Math.max(0, limit - used);

  return {
    allowed: remaining > 0 || limit === Infinity,
    remaining,
    limit,
    used,
  };
}

/**
 * Get quota info for API response
 */
export async function getQuotaInfo(userId: string): Promise<QuotaInfo> {
  const session = await getSession(userId);
  const plan = session?.plan || "free";
  const limit = getDailyLimit(plan);
  const used = await getDailyUsage(userId);
  const remaining = Math.max(0, limit - used);

  // Next reset at midnight
  const tomorrow = new Date();
  tomorrow.setDate(tomorrow.getDate() + 1);
  tomorrow.setHours(0, 0, 0, 0);

  return {
    user_id: userId,
    plan,
    daily_limit: limit,
    used_today: used,
    remaining,
    resets_at: tomorrow.toISOString(),
  };
}
```

---

## Task 4: 任务索引集合 — lib/task-index.ts

### 目标
提供Set索引操作，实现分批迭代和超时保护，避免smembers全量扫描。

### 依赖
- **引入包**：`@vercel/kv`
- **引入模块**：`lib/constants.ts`
- **使用常量**：`PENDING_PLATE_TASKS`, `PENDING_ENHANCE_TASKS`, `PENDING_VERIFY_TASKS`

### 函数签名

```typescript
import { kv } from "@vercel/kv";
import { SSCAN_TIMEOUT_MS, SSCAN_BATCH_SIZE } from "./constants";

/**
 * Add task to index collection
 */
export async function addToIndex(index: string, taskId: string): Promise<void> {
  await kv.sadd(index, taskId);
}

/**
 * Remove task from index collection
 */
export async function removeFromIndex(index: string, taskId: string): Promise<void> {
  await kv.srem(index, taskId);
}

/**
 * Get pending tasks using sscan with timeout protection [P0-E]
 * Avoids smembers which can timeout with large datasets
 * 
 * @param index - Index collection name
 * @param batchSize - Max tasks to return (default 100)
 * @param timeoutMs - Max time to spend iterating (default 5000ms)
 */
export async function getPendingTasksBatch(
  index: string,
  batchSize: number = SSCAN_BATCH_SIZE,
  timeoutMs: number = SSCAN_TIMEOUT_MS
): Promise<string[]> {
  const start = Date.now();
  let cursor = 0;
  const results: string[] = [];

  do {
    const [newCursor, members] = await kv.sscan(index, cursor, { count: batchSize });
    cursor = parseInt(newCursor as string, 10);
    results.push(...(Array.isArray(members) ? members : []));

    // Timeout protection
    if (Date.now() - start > timeoutMs) break;
  } while (cursor !== 0 && results.length < batchSize * 10);

  return results.slice(0, batchSize);
}

/**
 * Get all members (use only for small datasets)
 */
export async function getPendingTasks(index: string): Promise<string[]> {
  return await kv.smembers(index);
}

/**
 * Get index collection size
 */
export async function getIndexSize(index: string): Promise<number> {
  return await kv.scard(index);
}

/**
 * Check if task exists in index
 */
export async function isTaskInIndex(index: string, taskId: string): Promise<boolean> {
  return await kv.sismember(index, taskId);
}
```

---

## Task 5: Astrometry.net客户端 — lib/astrometry.ts

### 目标
封装Astrometry.net API调用：登录获取session、上传图片、轮询状态、获取校准数据。

### 依赖
- **引入包**：`@vercel/kv`
- **引入模块**：`lib/constants.ts`
- **使用常量**：`ASTROMETRY_BASE_URL`

### 函数签名

```typescript
import { ASTROMETRY_BASE_URL, ASTROMETRY_TIMEOUT, ASTROMETRY_UPLOAD_TIMEOUT } from "./constants";
import type { AstrometryData } from "./types";

interface AstrometrySession {
  session: string;
}

interface AstrometrySubmission {
  subid: string;
}

interface AstrometryJobStatus {
  status: "success" | "failure" | "solving";
  job_id?: number;
}

interface AstrometryJobInfo {
  job_id: number;
  status: string;
  objects_in_field: string[];
  calibration?: {
    ra: string;
    dec: string;
    pixscale: number;
    radius: number;
    orientation: number;
  };
}

/**
 * Login to Astrometry.net and get session key
 */
export async function astrometryLogin(): Promise<string> {
  const apiKey = process.env.ASTROMETRY_API_KEY;
  if (!apiKey) {
    throw new Error("ASTROMETRY_API_KEY is required");
  }

  const response = await fetch(`${ASTROMETRY_BASE_URL}/login`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ request_json: { apikey: apiKey } }),
    signal: AbortSignal.timeout(ASTROMETRY_TIMEOUT),
  });

  if (!response.ok) {
    throw new Error(`Astrometry login failed: ${response.status}`);
  }

  const data = await response.json() as AstrometrySession;
  return data.session;
}

/**
 * Upload image to Astrometry.net
 */
export async function astrometryUpload(
  session: string,
  file: File | Buffer
): Promise<string> {
  const formData = new FormData();
  
  const params: Record<string, string | number> = {
    session,
    allow_commercial_use: "n",
    allow_modifications: "n",
    publicly_visible: "n",
    scale_units: "arcsecperpix",
    scale_type: "ul",
    scale_lower: 0.5,
    scale_upper: 10,
    downsample_factor: 2,
  };

  formData.append("request_json", JSON.stringify(params));

  if (file instanceof File) {
    formData.append("file", file);
  } else {
    // Convert Buffer to Blob
    const blob = new Blob([file], { type: "image/jpeg" });
    formData.append("file", blob, "image.jpg");
  }

  const response = await fetch(`${ASTROMETRY_BASE_URL}/upload`, {
    method: "POST",
    body: formData,
    signal: AbortSignal.timeout(ASTROMETRY_UPLOAD_TIMEOUT),
  });

  if (!response.ok) {
    throw new Error(`Astrometry upload failed: ${response.status}`);
  }

  const data = await response.json() as AstrometrySubmission;
  
  if (!data.subid) {
    throw new Error("Astrometry upload returned no subid");
  }

  return data.subid;
}

/**
 * Get job ID from submission
 */
export async function astrometryGetSubmission(subid: string): Promise<number | null> {
  const session = await astrometryLogin();
  
  const response = await fetch(`${ASTROMETRY_BASE_URL}/submissions/${subid}`, {
    signal: AbortSignal.timeout(ASTROMETRY_TIMEOUT),
  });

  if (!response.ok) {
    return null;
  }

  const data = await response.json() as { jobs?: number[] };
  return data.jobs?.[0] ?? null;
}

/**
 * Poll Astrometry job status
 */
export async function astrometryPollJob(subid: string): Promise<{
  status: "success" | "failure" | "solving";
  jobId?: number;
}> {
  // First get job ID from submission
  const jobId = await astrometryGetSubmission(subid);
  
  if (!jobId) {
    return { status: "solving" };
  }

  const response = await fetch(`${ASTROMETRY_BASE_URL}/jobs/${jobId}`, {
    signal: AbortSignal.timeout(ASTROMETRY_TIMEOUT),
  });

  if (!response.ok) {
    return { status: "solving" };
  }

  const data = await response.json() as AstrometryJobStatus;
  
  if (data.status === "success") {
    return { status: "success", jobId };
  } else if (data.status === "failure") {
    return { status: "failure" };
  }

  return { status: "solving", jobId };
}

/**
 * Get job calibration info
 */
export async function astrometryGetInfo(jobId: number): Promise<AstrometryData> {
  const response = await fetch(`${ASTROMETRY_BASE_URL}/jobs/${jobId}/info`, {
    signal: AbortSignal.timeout(ASTROMETRY_TIMEOUT),
  });

  if (!response.ok) {
    throw new Error(`Astrometry get info failed: ${response.status}`);
  }

  const data = await response.json() as AstrometryJobInfo;
  
  return {
    ra: data.calibration?.ra ?? "unknown",
    dec: data.calibration?.dec ?? "unknown",
    objects: data.objects_in_field ?? [],
    pixscale: data.calibration?.pixscale ?? 0,
    radius: data.calibration?.radius ?? 0,
    orientation: data.calibration?.orientation ?? 0,
  };
}
```

---

## Task 6: Prompt构建器 — lib/prompt-builder.ts

### 目标
构建用于Gemini图像增强和验证的Prompt模板，支持变量填充。

### 依赖
- **引入包**：无
- **引入模块**：`lib/types.ts`, `lib/constants.ts`
- **使用类型**：`AstrometryData`

### 函数签名

```typescript
import type { AstrometryData } from "./types";
import { DEFAULT_STYLE } from "./constants";

/**
 * Build enhancement prompt with astrometry data
 */
export function buildEnhancementPrompt(
  astrometry: AstrometryData | null,
  style: "natural" | "vivid" = DEFAULT_STYLE
): string {
  const objectsInfo = astrometry?.objects?.length
    ? `Objects identified in this image: ${astrometry.objects.join(", ")}. These are well-known astronomical objects - enhance them according to their known characteristics.`
    : "";

  const coordsInfo = !astrometry?.objects?.length && astrometry?.ra
    ? `Image center coordinates: RA ${astrometry.ra}, DEC ${astrometry.dec}. Based on these coordinates, identify what celestial objects might be visible and enhance accordingly.`
    : "";

  const styleInstruction = style === "vivid"
    ? "Apply vivid but scientifically accurate color enhancement. Boost saturation moderately while maintaining authentic color relationships."
    : "Maintain natural, scientifically accurate appearance. Subtle enhancements only.";

  return `
You are a professional astrophotography image processor. Enhance this REAL telescope photograph following these exact steps:

STEP 1 - LIGHT POLLUTION REMOVAL: Remove all light pollution color casts (orange, green, brown gradients). Subtract background gradient pollution evenly. Sky background must become neutral dark, not tinted.

STEP 2 - NOISE REDUCTION: Apply advanced multi-scale denoising to the background sky while preserving ALL real star signals including the faintest ones. Aggressive on smooth background, gentle near star edges.

STEP 3 - NONLINEAR STRETCH: Calculate the statistical median pixel value of the image. Apply a nonlinear stretch (arcsinh-like) pivoting on this median balance point to expand mid-tone dynamic range and reveal faint nebula/galaxy signal while keeping bright stars from saturating. The stretch should be moderate.

STEP 4 - WHITE BALANCE & COLOR CALIBRATION: Auto-correct white balance. Restore accurate stellar colors by spectral type (hot O/B → blue-white, A → white, F/G → yellow-white, K → orange, M → red-orange).

Astrometry.net plate-solving calibration data:
- Center: RA ${astrometry?.ra || "unknown"}, DEC ${astrometry?.dec || "unknown"}
- FOV radius: ${astrometry?.radius || "unknown"}°
- Pixel scale: ${astrometry?.pixscale || "unknown"} arcsec/pixel
- Orientation: ${astrometry?.orientation || "unknown"}°
- Objects in field: ${astrometry?.objects?.join(", ") || "none identified"}

${objectsInfo}
${coordsInfo}

STEP 5 - SUPER-RESOLUTION & SHARPENING: Reconstruct at higher effective resolution. Tighten star profiles to be more point-like. Enhance fine detail without creating artifacts or halos around stars.

${styleInstruction}

ABSOLUTE SCIENTIFIC INTEGRITY RULES:
- Do NOT add any stars, nebulae, galaxies, or celestial objects NOT captured in the original telescope exposure
- Only enhance what is genuinely present in the image signal
- If identified objects are present as faint signal, enhance them realistically matching amateur astrophotography quality
- Do NOT render at Hubble/Space Telescope level detail - match what amateur equipment can capture
- Do NOT add diffraction spikes, lens flares, or optical artifacts not in the original
- Star count must remain consistent with the original
- Any color or detail enhancement must be derived from existing signal, not imagined
- This is a SCIENTIFIC OBSERVATION. Authenticity overrides aesthetics.
`.trim();
}
```

### 完整Prompt模板

#### 增强Prompt模板（英文原文，供Gemini直接使用）

```
You are a professional astrophotography image processor. Enhance this REAL telescope photograph following these exact steps:

STEP 1 - LIGHT POLLUTION REMOVAL: Remove all light pollution color casts (orange, green, brown gradients). Subtract background gradient pollution evenly. Sky background must become neutral dark, not tinted.

STEP 2 - NOISE REDUCTION: Apply advanced multi-scale denoising to the background sky while preserving ALL real star signals including the faintest ones. Aggressive on smooth background, gentle near star edges.

STEP 3 - NONLINEAR STRETCH: Calculate the statistical median pixel value of the image. Apply a nonlinear stretch (arcsinh-like) pivoting on this median balance point to expand mid-tone dynamic range and reveal faint nebula/galaxy signal while keeping bright stars from saturating. The stretch should be moderate.

STEP 4 - WHITE BALANCE & COLOR CALIBRATION: Auto-correct white balance. Restore accurate stellar colors by spectral type (hot O/B → blue-white, A → white, F/G → yellow-white, K → orange, M → red-orange).

Astrometry.net plate-solving calibration data:
- Center: RA {{ra_hms}}, DEC {{dec_dms}}
- FOV radius: {{radius_deg}}°
- Pixel scale: {{pixscale}} arcsec/pixel
- Orientation: {{orientation}}°
- Objects in field: {{objects_list}}

{{#each objects}}
If identified, enhance this object according to its known characteristics:
- {{name}}: {{type}} with {{magnitude}} magnitude
- Typical colors: {{typical_colors}}
{{/each}}

STEP 5 - SUPER-RESOLUTION & SHARPENING: Reconstruct at higher effective resolution. Tighten star profiles to be more point-like. Enhance fine detail without creating artifacts or halos around stars.

{{style_instruction}}

ABSOLUTE SCIENTIFIC INTEGRITY RULES:
- Do NOT add any stars, nebulae, galaxies, or celestial objects NOT captured in the original telescope exposure
- Only enhance what is genuinely present in the image signal
- If identified objects are present as faint signal, enhance them realistically matching amateur astrophotography quality
- Do NOT render at Hubble/Space Telescope level detail - match what amateur equipment can capture
- Do NOT add diffraction spikes, lens flares, or optical artifacts not in the original
- Star count must remain consistent with the original
- Any color or detail enhancement must be derived from existing signal, not imagined
- This is a SCIENTIFIC OBSERVATION. Authenticity overrides aesthetics.
```

**变量说明**：
- `{{ra_hms}}`: 天体赤经，格式如 "05h 35m 27.8s"
- `{{dec_dms}}`: 天体赤纬，格式如 "+22° 02' 59.1\""
- `{{radius_deg}}`: 视场半径（度）
- `{{pixscale}}`: 像素尺度（角秒/像素）
- `{{orientation}}`: 方向角（度）
- `{{objects_list}}`: 视场中的天体列表，逗号分隔
- `{{style_instruction}}`: 风格指令，natural或vivid

#### 验证Prompt模板（英文原文，供Gemini直接使用）

```
You are an expert astronomical image analyst. Verify the scientific authenticity of an enhanced astrophotography image by comparing it with the original.

IMAGE 1: The ORIGINAL raw telescope photograph
IMAGE 2: The ENHANCED version after processing

Perform these verification checks:

CHECK 1 - STAR COUNT INTEGRITY
Count approximate visible stars in both images. Enhanced must NOT contain significantly more stars. Report: original ~N stars, enhanced ~N stars, PASS/FAIL.

CHECK 2 - STAR POSITION ACCURACY
Stars in enhanced image must appear at same positions as original. No displacement, duplication, or removal. Report: PASS/FAIL.

CHECK 3 - NO FABRICATED CELESTIAL OBJECTS
Examine whether enhanced image contains nebulae, galaxies, diffuse emission, or other structures NOT in the original. Even if a known object genuinely exists in this sky region, if the original did not capture it, the enhanced must NOT fabricate it. Report: PASS/FAIL with list.

CHECK 4 - NO FABRICATED OPTICAL ARTIFACTS
Check for diffraction spikes, lens flares, satellite trails not in original. Report: PASS/FAIL.

CHECK 5 - COLOR AUTHENTICITY
Star colors match spectral types. Nebular emission colors match known lines (H-alpha → reddish-pink, OIII → blue-green, SII → deep red). No implausible colors. Report: PASS/FAIL.

CHECK 6 - DYNAMIC RANGE REALISM
Brightness and contrast enhancement realistic for amateur equipment, not exaggerated to space telescope level. Report: PASS/FAIL.

Plate-solving context:
- Center: RA {{ra_hms}}, DEC {{dec_dms}}
- Objects in field: {{objects_list}}

OVERALL VERDICT:
- AUTHENTIC: Enhancement preserves scientific integrity
- QUESTIONABLE: Some issues detected, list them
- FABRICATED: Significant fabrication, image cannot be trusted

Output as JSON:
{
  "authentic": true/false,
  "checks": {
    "star_count": {"pass": bool, "detail": "..."},
    "star_position": {"pass": bool, "detail": "..."},
    "no_fabricated_objects": {"pass": bool, "detail": "..."},
    "no_artifacts": {"pass": bool, "detail": "..."},
    "color_authenticity": {"pass": bool, "detail": "..."},
    "dynamic_range": {"pass": bool, "detail": "..."}
  },
  "issues": [],
  "verdict": "AUTHENTIC|QUESTIONABLE|FABRICATED"
}
```

**变量说明**：
- `{{ra_hms}}`: 天体赤经
- `{{dec_dms}}`: 天体赤纬
- `{{objects_list}}`: 视场中的天体列表

---

## Task 7: Gemini图像增强 — lib/gemini-enhance.ts

### 目标
调用Gemini进行图像增强，生成3张增强版本供用户选择。

### 依赖
- **引入包**：`@google/generative-ai`
- **引入模块**：`lib/constants.ts`, `lib/prompt-builder.ts`
- **使用常量**：`GEMINI_MODEL_ENHANCE`, `GEMINI_ENHANCE_TIMEOUT`, `ENHANCED_VERSION_COUNT`

### 函数签名

```typescript
import { GoogleGenerativeAI } from "@google/generative-ai";
import { buildEnhancementPrompt } from "./prompt-builder";
import { GEMINI_MODEL_ENHANCE, GEMINI_ENHANCE_TIMEOUT, ENHANCED_VERSION_COUNT } from "./constants";
import type { AstrometryData } from "./types";

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);

interface EnhancementResult {
  images: Buffer[];
  verification?: VerificationResult;
}

interface VerificationResult {
  authentic: boolean;
  checks?: Record<string, { pass: boolean; detail?: string }>;
  issues?: string[];
  verdict?: string;
}

/**
 * Enhance image with Gemini - generates multiple versions
 */
export async function enhanceWithGemini(
  imageBuffer: Buffer,
  astrometry: AstrometryData | null,
  style: "natural" | "vivid" = "natural"
): Promise<EnhancementResult> {
  const model = genAI.getGenerativeModel({
    model: GEMINI_MODEL_ENHANCE,
    generationConfig: {
      responseModalities: ["TEXT", "IMAGE"],
      temperature: 1.0,
    },
  });

  const prompt = buildEnhancementPrompt(astrometry, style);
  const imageBase64 = imageBuffer.toString("base64");

  // Generate multiple versions
  const images: Buffer[] = [];
  const verificationResults: VerificationResult[] = [];

  for (let i = 0; i < ENHANCED_VERSION_COUNT; i++) {
    const result = await Promise.race([
      model.generateContent([
        { text: `${prompt}\n\nGenerate version ${i + 1} of ${ENHANCED_VERSION_COUNT}.` },
        { inlineData: { mimeType: "image/jpeg", data: imageBase64 } },
      ]),
      new Promise<never>((_, reject) =>
        setTimeout(() => reject(new Error("Gemini timeout")), GEMINI_ENHANCE_TIMEOUT * 1000)
      ),
    ]);

    // Parse response
    for (const part of result.response.candidates?.[0]?.content?.parts || []) {
      if (part.inlineData) {
        images.push(Buffer.from(part.inlineData.data, "base64"));
      }
      if (part.text) {
        try {
          const parsed = JSON.parse(part.text);
          if (parsed.self_verification) {
            verificationResults.push({
              authentic: parsed.self_verification.star_count_preserved &&
                        parsed.self_verification.no_fabricated_objects,
              verdict: parsed.self_verification.quality_score > 70 ? "AUTHENTIC" : "QUESTIONABLE",
            });
          }
        } catch {
          // Not JSON, ignore
        }
      }
    }
  }

  // Return best version or all versions
  return {
    images,
    verification: verificationResults[0],
  };
}

/**
 * Enhance with inline verification
 */
export async function enhanceWithGeminiAndVerify(
  imageBuffer: Buffer,
  astrometry: AstrometryData | null,
  style: "natural" | "vivid" = "natural"
): Promise<EnhancementResult> {
  const model = genAI.getGenerativeModel({
    model: GEMINI_MODEL_ENHANCE,
    generationConfig: {
      responseModalities: ["TEXT", "IMAGE"],
      temperature: 1.0,
    },
  });

  const prompt = buildEnhancementPrompt(astrometry, style);
  const imageBase64 = imageBuffer.toString("base64");

  // Enhanced prompt with inline verification request
  const enhancedPrompt = `${prompt}

SELF-VERIFICATION (output after the image):
After generating the enhanced image, perform a self-check and output a JSON report:

{
  "self_verification": {
    "star_count_preserved": true/false,
    "no_fabricated_objects": true/false,
    "color_authentic": true/false,
    "quality_score": 0-100
  }
}`;

  const result = await Promise.race([
    model.generateContent([
      { text: enhancedPrompt },
      { inlineData: { mimeType: "image/jpeg", data: imageBase64 } },
    ]),
    new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error("Gemini timeout")), GEMINI_ENHANCE_TIMEOUT * 1000)
    ),
  ]);

  const images: Buffer[] = [];
  let verification: VerificationResult | undefined;

  for (const part of result.response.candidates?.[0]?.content?.parts || []) {
    if (part.inlineData) {
      images.push(Buffer.from(part.inlineData.data, "base64"));
    }
    if (part.text) {
      try {
        const parsed = JSON.parse(part.text);
        if (parsed.self_verification) {
          verification = {
            authentic: parsed.self_verification.star_count_preserved &&
                      parsed.self_verification.no_fabricated_objects,
            checks: {
              star_count: { pass: parsed.self_verification.star_count_preserved },
              no_fabricated_objects: { pass: parsed.self_verification.no_fabricated_objects },
              color_authenticity: { pass: parsed.self_verification.color_authentic },
            },
            verdict: parsed.self_verification.quality_score > 70 ? "AUTHENTIC" : "QUESTIONABLE",
          };
        }
      } catch {
        // Not JSON, ignore
      }
    }
  }

  return { images, verification };
}
```

---

## Task 8: Gemini二次验证 — lib/gemini-verify.ts

### 目标
独立调用Gemini对增强图像进行二次验证。

### 依赖
- **引入包**：`@google/generative-ai`
- **引入模块**：`lib/constants.ts`
- **使用类型**：`VerificationResult`, `AstrometryData`

### 函数签名

```typescript
import { GoogleGenerativeAI } from "@google/generative-ai";
import { GEMINI_MODEL_VERIFY, GEMINI_VERIFY_TIMEOUT } from "./constants";
import type { VerificationResult, AstrometryData } from "./types";

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);

/**
 * Verify enhanced image against original
 */
export async function verifyWithGemini(
  originalBuffer: Buffer,
  enhancedBuffer: Buffer,
  astrometry?: AstrometryData
): Promise<VerificationResult> {
  const model = genAI.getGenerativeModel({
    model: GEMINI_MODEL_VERIFY,
    generationConfig: {
      responseModalities: ["TEXT"],
    },
  });

  const originalBase64 = originalBuffer.toString("base64");
  const enhancedBase64 = enhancedBuffer.toString("base64");

  const prompt = `You are an expert astronomical image analyst. Verify the scientific authenticity of an enhanced astrophotography image by comparing it with the original.

IMAGE 1: The ORIGINAL raw telescope photograph
IMAGE 2: The ENHANCED version after processing

Perform these verification checks:

CHECK 1 - STAR COUNT INTEGRITY
Count approximate visible stars in both images. Enhanced must NOT contain significantly more stars. Report: original ~N stars, enhanced ~N stars, PASS/FAIL.

CHECK 2 - STAR POSITION ACCURACY
Stars in enhanced image must appear at same positions as original. No displacement, duplication, or removal. Report: PASS/FAIL.

CHECK 3 - NO FABRICATED CELESTIAL OBJECTS
Examine whether enhanced image contains nebulae, galaxies, diffuse emission, or other structures NOT in the original. Even if a known object genuinely exists in this sky region, if the original did not capture it, the enhanced must NOT fabricate it. Report: PASS/FAIL with list.

CHECK 4 - NO FABRICATED OPTICAL ARTIFACTS
Check for diffraction spikes, lens flares, satellite trails not in original. Report: PASS/FAIL.

CHECK 5 - COLOR AUTHENTICITY
Star colors match spectral types. Nebular emission colors match known lines (H-alpha → reddish-pink, OIII → blue-green, SII → deep red). No implausible colors. Report: PASS/FAIL.

CHECK 6 - DYNAMIC RANGE REALISM
Brightness and contrast enhancement realistic for amateur equipment, not exaggerated to space telescope level. Report: PASS/FAIL.

Plate-solving context:
- Center: RA ${astrometry?.ra || "unknown"}, DEC ${astrometry?.dec || "unknown"}
- Objects in field: ${astrometry?.objects?.join(", ") || "none identified"}

OVERALL VERDICT:
- AUTHENTIC: Enhancement preserves scientific integrity
- QUESTIONABLE: Some issues detected, list them
- FABRICATED: Significant fabrication, image cannot be trusted

Output as JSON:
{
  "authentic": true/false,
  "checks": {
    "star_count": {"pass": bool, "detail": "..."},
    "star_position": {"pass": bool, "detail": "..."},
    "no_fabricated_objects": {"pass": bool, "detail": "..."},
    "no_artifacts": {"pass": bool, "detail": "..."},
    "color_authenticity": {"pass": bool, "detail": "..."},
    "dynamic_range": {"pass": bool, "detail": "..."}
  },
  "issues": [],
  "verdict": "AUTHENTIC|QUESTIONABLE|FABRICATED"
}`;

  const result = await Promise.race([
    model.generateContent([
      { text: prompt },
      { inlineData: { mimeType: "image/jpeg", data: originalBase64 } },
      { inlineData: { mimeType: "image/png", data: enhancedBase64 } },
    ]),
    new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error("Gemini verify timeout")), GEMINI_VERIFY_TIMEOUT * 1000)
    ),
  ]);

  const text = result.response.text();
  
  try {
    // Extract JSON from response
    const jsonMatch = text.match(/\{[\s\S]*\}/);
    if (jsonMatch) {
      return JSON.parse(jsonMatch[0]) as VerificationResult;
    }
  } catch {
    // Fall through to default
  }

  // Default fallback
  return {
    authentic: false,
    issues: ["Failed to parse verification result"],
    verdict: "QUESTIONABLE",
  };
}
```

---

## Task 9: 文件存储 — lib/file-store.ts

### 目标
封装Vercel Blob操作，实现文件上传和URL生成。

### 依赖
- **引入包**：`@vercel/blob`
- **引入模块**：`lib/constants.ts`
- **使用常量**：`MAX_FILE_SIZE`, `ALLOWED_MIME_TYPES`

### 函数签名

```typescript
import { put, del } from "@vercel/blob";
import { MAX_FILE_SIZE, ALLOWED_MIME_TYPES } from "./constants";

export interface FileValidationResult {
  valid: boolean;
  error?: string;
}

/**
 * Validate uploaded file
 */
export function validateFile(file: File): FileValidationResult {
  // Check size
  if (file.size > MAX_FILE_SIZE) {
    return {
      valid: false,
      error: "FILE_TOO_LARGE",
    };
  }

  // Check mime type
  if (!ALLOWED_MIME_TYPES.includes(file.type as typeof ALLOWED_MIME_TYPES[number])) {
    return {
      valid: false,
      error: "UNSUPPORTED_FORMAT",
    };
  }

  return { valid: true };
}

/**
 * Upload original image to Blob
 */
export async function uploadOriginalImage(
  taskId: string,
  file: File
): Promise<{ url: string; pathname: string }> {
  const blob = await put(`original/${taskId}`, file, {
    access: "public",
    contentType: file.type,
  });

  return {
    url: blob.url,
    pathname: blob.pathname,
  };
}

/**
 * Upload enhanced image to Blob
 */
export async function uploadEnhancedImage(
  taskId: string,
  imageBuffer: Buffer,
  version?: number
): Promise<{ url: string; pathname: string }> {
  const filename = version !== undefined
    ? `enhanced/${taskId}_v${version}.png`
    : `enhanced/${taskId}.png`;

  const blob = await put(filename, imageBuffer, {
    access: "public",
    contentType: "image/png",
  });

  return {
    url: blob.url,
    pathname: blob.pathname,
  };
}

/**
 * Delete file from Blob
 */
export async function deleteFile(pathname: string): Promise<void> {
  await del(pathname);
}
```

---

## Task 10: API路由 — api/auth/register.ts + login.ts

### 目标
实现用户注册和登录接口。

### 依赖
- **引入包**：`@vercel/kv`, `jose` (for password hashing simulation)
- **引入模块**：`lib/auth.ts`, `lib/session-store.ts`, `middleware/auth.ts`
- **使用类型**：`JWTPayload`, `UserPlan`

### API规格

#### POST /api/auth/register

**请求**：
```json
{
  "email": "user@example.com",
  "password": "secure_password"
}
```

**响应（成功）**：
```json
{
  "success": true,
  "user_id": "usr_abc123def456",
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "plan": "free",
  "daily_quota": 3
}
```

**响应（邮箱已存在）**：
```json
{
  "success": false,
  "error": "EMAIL_EXISTS",
  "user_message": "该邮箱已注册，请直接登录",
  "code": 400
}
```

**完整代码**：

```typescript
import { kv } from "@vercel/kv";
import { randomBytes } from "crypto";
import { generateToken } from "../../lib/auth";
import { createSession } from "../../lib/session-store";
import { USER_PREFIX } from "../../lib/constants";

export default async function handler(req: Request) {
  if (req.method !== "POST") {
    return Response.json({ error: "METHOD_NOT_ALLOWED" }, { status: 405 });
  }

  const body = await req.json().catch(() => null);
  const { email, password } = body || {};

  // Validate input
  if (!email || !password) {
    return Response.json(
      {
        success: false,
        error: "INVALID_INPUT",
        user_message: "请提供邮箱和密码",
        code: 400,
      },
      { status: 400 }
    );
  }

  // Check email format
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!emailRegex.test(email)) {
    return Response.json(
      {
        success: false,
        error: "INVALID_EMAIL",
        user_message: "请输入有效的邮箱地址",
        code: 400,
      },
      { status: 400 }
    );
  }

  // Check password length
  if (password.length < 8) {
    return Response.json(
      {
        success: false,
        error: "WEAK_PASSWORD",
        user_message: "密码长度至少8位",
        code: 400,
      },
      { status: 400 }
    );
  }

  // Check if email exists
  const userKey = `${USER_PREFIX}email:${email.toLowerCase()}`;
  const existingUser = await kv.get(userKey);
  if (existingUser) {
    return Response.json(
      {
        success: false,
        error: "EMAIL_EXISTS",
        user_message: "该邮箱已注册，请直接登录",
        code: 400,
      },
      { status: 400 }
    );
  }

  // Create user
  const userId = `usr_${randomBytes(12).toString("hex")}`;
  const passwordHash = await hashPassword(password); // Simple hash for demo
  
  const user = {
    user_id: userId,
    email: email.toLowerCase(),
    password_hash: passwordHash,
    plan: "free" as UserPlan,
    created_at: Date.now(),
  };

  // Store user
  await kv.set(`${USER_PREFIX}user:${userId}`, user);
  await kv.set(userKey, userId);

  // Create session and generate token
  await createSession(userId, "free");
  const token = await generateToken(userId, "free", email);

  return Response.json({
    success: true,
    user_id: userId,
    token,
    plan: "free",
    daily_quota: 3,
  });
}

// Simple password hashing (use bcrypt in production)
async function hashPassword(password: string): Promise<string> {
  const encoder = new TextEncoder();
  const data = encoder.encode(password + process.env.JWT_SECRET);
  const hashBuffer = await crypto.subtle.digest("SHA-256", data);
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  return hashArray.map(b => b.toString(16).padStart(2, "0")).join("");
}
```

#### POST /api/auth/login

**请求**：
```json
{
  "email": "user@example.com",
  "password": "secure_password"
}
```

**响应（成功）**：
```json
{
  "success": true,
  "user_id": "usr_abc123def456",
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "plan": "free",
  "daily_quota": 8
}
```

**响应（密码错误）**：
```json
{
  "success": false,
  "error": "INVALID_CREDENTIALS",
  "user_message": "邮箱或密码错误",
  "code": 401
}
```

**完整代码**：

```typescript
import { kv } from "@vercel/kv";
import { generateToken } from "../../lib/auth";
import { createSession, getQuotaInfo } from "../../lib/session-store";
import { USER_PREFIX } from "../../lib/constants";

export default async function handler(req: Request) {
  if (req.method !== "POST") {
    return Response.json({ error: "METHOD_NOT_ALLOWED" }, { status: 405 });
  }

  const body = await req.json().catch(() => null);
  const { email, password } = body || {};

  if (!email || !password) {
    return Response.json(
      {
        success: false,
        error: "INVALID_INPUT",
        user_message: "请提供邮箱和密码",
        code: 400,
      },
      { status: 400 }
    );
  }

  // Find user by email
  const userKey = `${USER_PREFIX}email:${email.toLowerCase()}`;
  const userId = await kv.get<string>(userKey);
  
  if (!userId) {
    return Response.json(
      {
        success: false,
        error: "INVALID_CREDENTIALS",
        user_message: "邮箱或密码错误",
        code: 401,
      },
      { status: 401 }
    );
  }

  // Get user
  const user = await kv.get(`${USER_PREFIX}user:${userId}`);
  if (!user) {
    return Response.json(
      {
        success: false,
        error: "INVALID_CREDENTIALS",
        user_message: "邮箱或密码错误",
        code: 401,
      },
      { status: 401 }
    );
  }

  // Verify password
  const passwordHash = await hashPassword(password);
  if (passwordHash !== user.password_hash) {
    return Response.json(
      {
        success: false,
        error: "INVALID_CREDENTIALS",
        user_message: "邮箱或密码错误",
        code: 401,
      },
      { status: 401 }
    );
  }

  // Create session and generate token
  await createSession(userId, user.plan);
  const token = await generateToken(userId, user.plan, user.email);
  const quota = await getQuotaInfo(userId);

  return Response.json({
    success: true,
    user_id: userId,
    token,
    plan: user.plan,
    daily_quota: quota.remaining,
  });
}

async function hashPassword(password: string): Promise<string> {
  const encoder = new TextEncoder();
  const data = encoder.encode(password + process.env.JWT_SECRET);
  const hashBuffer = await crypto.subtle.digest("SHA-256", data);
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  return hashArray.map(b => b.toString(16).padStart(2, "0")).join("");
}
```

---

## Task 11: API路由 — api/upload.ts

### 目标
处理图片上传，触发Astrometry plate-solving。

### 依赖
- **引入包**：`@vercel/kv`, `@vercel/blob`
- **引入模块**：`middleware/auth.ts`, `lib/session-store.ts`, `lib/task-index.ts`, `lib/astrometry.ts`, `lib/file-store.ts`, `lib/constants.ts`

### API规格

**端点**：POST /api/upload

**Header**：
```
Authorization: Bearer <JWT>
Content-Type: multipart/form-data
```

**请求Body**：
```
file: <图片文件>  (JPG/PNG/FITS, 最大20MB)
```

**响应（成功）**：
```json
{
  "task_id": "task_1704067200000_abc123def456",
  "status": "uploaded",
  "message": "图片已上传，开始plate-solving",
  "estimated_wait_seconds": 180
}
```

**响应（未认证）**：
```json
{
  "error": "UNAUTHORIZED",
  "user_message": "请先登录后使用",
  "code": 401
}
```

**响应（配额超限）**：
```json
{
  "error": "QUOTA_EXCEEDED",
  "user_message": "今日使用次数已用完",
  "code": 429
}
```

**响应（文件过大）**：
```json
{
  "error": "FILE_TOO_LARGE",
  "user_message": "图片文件过大，请上传20MB以下的图片",
  "code": 400
}
```

**完整代码**：

```typescript
import { kv } from "@vercel/kv";
import { randomBytes } from "crypto";
import { authMiddleware, getUserId } from "../middleware/auth";
import { checkQuota, incrementDailyUsage } from "../lib/session-store";
import { addToIndex } from "../lib/task-index";
import { astrometryLogin, astrometryUpload } from "../lib/astrometry";
import { validateFile, uploadOriginalImage } from "../lib/file-store";
import { PENDING_PLATE_TASKS } from "../lib/constants";

export default async function handler(req: Request) {
  // 1. Authentication check
  const authError = await authMiddleware(req);
  if (authError) return authError;

  const userId = getUserId(req);
  if (!userId) {
    return Response.json(
      { error: "UNAUTHORIZED", user_message: "请先登录后使用", code: 401 },
      { status: 401 }
    );
  }

  // 2. Quota check
  const quota = await checkQuota(userId);
  if (!quota.allowed) {
    return Response.json(
      {
        error: "QUOTA_EXCEEDED",
        user_message: `今日使用次数已用完，${quota.used}次已用完`,
        code: 429,
      },
      { status: 429 }
    );
  }

  // 3. Parse multipart upload
  const formData = await req.formData();
  const file = formData.get("file") as File | null;

  if (!file) {
    return Response.json(
      {
        error: "INVALID_INPUT",
        user_message: "请上传图片文件",
        code: 400,
      },
      { status: 400 }
    );
  }

  // 4. Validate file
  const validation = validateFile(file);
  if (!validation.valid) {
    const userMessages: Record<string, string> = {
      FILE_TOO_LARGE: "图片文件过大，请上传20MB以下的图片",
      UNSUPPORTED_FORMAT: "暂不支持该图片格式，请上传JPG/PNG/FITS格式",
    };
    return Response.json(
      {
        error: validation.error,
        user_message: userMessages[validation.error!] || "文件校验失败",
        code: 400,
      },
      { status: 400 }
    );
  }

  // 5. Generate secure task ID [P1-4]
  const taskId = `task_${Date.now()}_${randomBytes(16).toString("hex").slice(0, 24)}`;

  // 6. Upload to Vercel Blob
  const { url: originalUrl } = await uploadOriginalImage(taskId, file);

  // 7. Login to Astrometry and upload
  try {
    const session = await astrometryLogin();
    const subid = await astrometryUpload(session, file);

    // 8. Create task record
    await kv.set(taskId, {
      task_id: taskId,
      status: "plate_solving",
      subid,
      originalUrl,
      userId,
      createdAt: Date.now(),
    });

    // 9. Add to index collection
    await addToIndex(PENDING_PLATE_TASKS, taskId);

    // 10. Increment quota usage
    await incrementDailyUsage(userId);

    return Response.json({
      task_id: taskId,
      status: "uploaded",
      message: "图片已上传，开始plate-solving",
      estimated_wait_seconds: 180,
    });
  } catch (error) {
    console.error("Astrometry upload failed:", error);
    return Response.json(
      {
        error: "ASTROMETRY_UPLOAD_FAILED",
        user_message: "图片上传失败，请检查网络后重试",
        code: 500,
      },
      { status: 500 }
    );
  }
}
```

---

## Task 12: API路由 — api/status/[taskId].ts

### 目标
查询任务处理状态。

### 依赖
- **引入包**：`@vercel/kv`
- **引入模块**：`middleware/auth.ts`, `lib/types.ts`

### API规格

**端点**：GET /api/status/:taskId

**Header**：
```
Authorization: Bearer <JWT>
```

**响应（处理中）**：
```json
{
  "task_id": "task_abc123",
  "status": "plate_solving",
  "progress": {
    "step": "astrometry",
    "detail": "Astrometry处理中，已等待45秒"
  },
  "estimated_wait_seconds": 120
}
```

**响应（已完成）**：
```json
{
  "task_id": "task_abc123",
  "status": "plate_solved",
  "progress": {
    "step": "plate_solving",
    "detail": "星图识别已完成"
  },
  "estimated_wait_seconds": 0,
  "astrometry": {
    "ra": "05h 35m 27.8s",
    "dec": "+22° 02' 59.1\"",
    "objects": ["M 1", "NGC 1952"],
    "pixscale": 3.30,
    "fov_radius": 0.70,
    "orientation": 316.15
  }
}
```

**完整代码**：

```typescript
import { kv } from "@vercel/kv";
import { authMiddleware, getUserId } from "../../middleware/auth";

export default async function handler(req: Request, { params }: { params: { taskId: string } }) {
  // 1. Authentication check
  const authError = await authMiddleware(req);
  if (authError) return authError;

  const userId = getUserId(req);
  const { taskId } = params;

  // 2. Get task [P0-B: ownership check]
  const task = await kv.get(taskId);
  if (!task || task.userId !== userId) {
    return Response.json(
      {
        error: "TASK_NOT_FOUND",
        user_message: "任务不存在或已过期",
        code: 404,
      },
      { status: 404 }
    );
  }

  // 3. Calculate estimated wait time
  const estimatedWait = calculateEstimatedWait(task.status, task.createdAt);

  // 4. Build response
  const response: Record<string, unknown> = {
    task_id: taskId,
    status: task.status,
    progress: {
      step: getStepName(task.status),
      detail: getStepDetail(task),
    },
    estimated_wait_seconds: estimatedWait,
  };

  // 5. Add astrometry data if available
  if (task.astrometry) {
    response.astrometry = {
      ra: task.astrometry.ra,
      dec: task.astrometry.dec,
      objects: task.astrometry.objects,
      pixscale: task.astrometry.pixscale,
      fov_radius: task.astrometry.radius,
      orientation: task.astrometry.orientation,
    };
  }

  return Response.json(response);
}

function calculateEstimatedWait(status: string, createdAt: number): number {
  const elapsed = Date.now() - createdAt;
  switch (status) {
    case "plate_solving":
      return Math.max(0, 300 - elapsed / 1000);
    case "enhancing":
      return Math.max(0, 60 - elapsed / 1000);
    case "verifying":
      return Math.max(0, 30 - elapsed / 1000);
    default:
      return 0;
  }
}

function getStepName(status: string): string {
  const steps: Record<string, string> = {
    uploaded: "upload",
    plate_solving: "astrometry",
    plate_solved: "plate_solving",
    plate_failed: "plate_solving",
    enhancing: "enhancement",
    verifying: "verification",
    completed: "done",
    verification_failed: "verification",
  };
  return steps[status] || status;
}

function getStepDetail(task: Record<string, unknown>): string {
  const elapsed = Math.floor((Date.now() - (task.createdAt as number)) / 1000);
  const status = task.status as string;

  const details: Record<string, string> = {
    uploaded: "图片已上传",
    plate_solving: `星图识别中，已等待${elapsed}秒`,
    plate_solved: "星图识别已完成",
    plate_failed: "星图识别失败",
    enhancing: "图像增强中",
    verifying: "二次验证中",
    completed: "处理完成",
    verification_failed: "验证未通过",
  };

  return details[status] || status;
}
```

---

## Task 13: API路由 — api/enhance/[taskId].ts

### 目标
触发图像增强任务。

### 依赖
- **引入包**：`@vercel/kv`, `@vercel/blob`
- **引入模块**：`middleware/auth.ts`, `lib/session-store.ts`, `lib/task-index.ts`, `lib/gemini-enhance.ts`, `lib/prompt-builder.ts`, `lib/file-store.ts`, `lib/constants.ts`

### API规格

**端点**：POST /api/enhance/:taskId

**Header**：
```
Authorization: Bearer <JWT>
Content-Type: application/json
```

**请求Body**（可选）：
```json
{
  "style": "natural",
  "denoise_level": 3,
  "stretch_intensity": 3,
  "skip_verification": false
}
```

**响应（成功）**：
```json
{
  "task_id": "task_abc123",
  "status": "enhancing",
  "message": "开始Gemini图像增强"
}
```

**完整代码**：

```typescript
import { kv } from "@vercel/kv";
import { authMiddleware, getUserId } from "../../middleware/auth";
import { getSession } from "../../lib/session-store";
import { addToIndex } from "../../lib/task-index";
import { enhanceWithGemini, enhanceWithGeminiAndVerify } from "../../lib/gemini-enhance";
import { buildEnhancementPrompt } from "../../lib/prompt-builder";
import { uploadEnhancedImage } from "../../lib/file-store";
import { PENDING_ENHANCE_TASKS, DEFAULT_STYLE } from "../../lib/constants";

export default async function handler(req: Request, { params }: { params: { taskId: string } }) {
  // 1. Authentication check
  const authError = await authMiddleware(req);
  if (authError) return authError;

  const userId = getUserId(req);
  const { taskId } = params;

  // 2. Get task [P0-B: ownership check]
  const task = await kv.get(taskId);
  if (!task || task.userId !== userId) {
    return Response.json(
      {
        error: "TASK_NOT_FOUND",
        user_message: "任务不存在或已过期",
        code: 404,
      },
      { status: 404 }
    );
  }

  // 3. Status check: must be plate_solved or plate_failed
  if (!["plate_solved", "plate_failed"].includes(task.status)) {
    return Response.json(
      {
        error: "INVALID_STATUS_TRANSITION",
        user_message: "当前状态不支持该操作",
        code: 400,
      },
      { status: 400 }
    );
  }

  // 4. Parse request body
  const body = await req.json().catch(() => ({}));
  const style = body.style || DEFAULT_STYLE;
  const skipVerification = body.skip_verification === true;

  // 5. Check if user can skip verification [v2.1-fixed: pro/enterprise only]
  const session = await getSession(userId);
  const userCanSkip = session?.plan === "pro" || session?.plan === "enterprise";
  const shouldSkip = skipVerification && userCanSkip;

  // 6. Update status
  await kv.set(taskId, {
    ...task,
    status: "enhancing",
    style,
    updatedAt: Date.now(),
  });

  try {
    // 7. Get original image
    const imageResponse = await fetch(task.originalUrl);
    const imageBuffer = Buffer.from(await imageResponse.arrayBuffer());

    // 8. Call Gemini enhancement
    const result = shouldSkip
      ? await enhanceWithGemini(imageBuffer, task.astrometry || null, style)
      : await enhanceWithGeminiAndVerify(imageBuffer, task.astrometry || null, style);

    // 9. Upload enhanced images (store multiple versions)
    const enhancedUrls: string[] = [];
    for (let i = 0; i < result.images.length; i++) {
      const blob = await uploadEnhancedImage(`${taskId}_v${i}`, result.images[i], i);
      enhancedUrls.push(blob.url);
    }

    // 10. Update task with first enhanced image URL and verification result
    await kv.set(taskId, {
      ...task,
      status: shouldSkip ? "completed" : "verifying",
      enhancedUrl: enhancedUrls[0],
      enhancedUrls,
      verification: result.verification,
      updatedAt: Date.now(),
    });

    // 11. Add to verify queue if not skipped
    if (!shouldSkip) {
      await addToIndex(PENDING_ENHANCE_TASKS, taskId);
    }

    return Response.json({
      task_id: taskId,
      status: shouldSkip ? "completed" : "enhancing",
      message: shouldSkip ? "图像增强完成" : "开始Gemini图像增强",
    });
  } catch (error) {
    console.error("Enhancement failed:", error);
    
    // Revert status
    await kv.set(taskId, {
      ...task,
      status: task.status,
      error: "GEMINI_ENHANCE_FAILED",
    });

    return Response.json(
      {
        error: "GEMINI_ENHANCE_FAILED",
        user_message: "图像增强失败，请稍后重试",
        code: 504,
      },
      { status: 504 }
    );
  }
}
```

---

## Task 14: API路由 — api/result/[taskId].ts

### 目标
获取增强结果。

### 依赖
- **引入包**：`@vercel/kv`
- **引入模块**：`middleware/auth.ts`

### API规格

**端点**：GET /api/result/:taskId

**Header**：
```
Authorization: Bearer <JWT>
```

**响应（成功完成）**：
```json
{
  "task_id": "task_abc123",
  "status": "completed",
  "original_url": "/api/file/task_abc123_original.jpg",
  "enhanced_url": "/api/file/task_abc123_enhanced.png",
  "annotated_info": {
    "objects": ["M 1", "NGC 1952"],
    "constellation": "Taurus",
    "coordinates": "RA 05h 35m 28s, DEC +22° 03'"
  },
  "verification": {
    "authentic": true,
    "checks": {
      "star_count": "PASS",
      "star_position": "PASS",
      "no_fabricated_objects": "PASS",
      "no_artifacts": "PASS",
      "color_authenticity": "PASS",
      "dynamic_range": "PASS"
    }
  }
}
```

**响应（验证失败）**：
```json
{
  "task_id": "task_abc123",
  "status": "verification_failed",
  "enhanced_url": "/api/file/task_abc123_enhanced.png",
  "error": "VERIFICATION_FAILED",
  "user_message": "检测到增强图中存在不真实的星点，建议调整参数后重试",
  "verification": {
    "authentic": false,
    "issues": ["检测到2颗额外星点", "星云颜色偏饱和"],
    "retry_available": true
  }
}
```

**完整代码**：

```typescript
import { kv } from "@vercel/kv";
import { authMiddleware, getUserId } from "../../middleware/auth";

export default async function handler(req: Request, { params }: { params: { taskId: string } }) {
  // 1. Authentication check
  const authError = await authMiddleware(req);
  if (authError) return authError;

  const userId = getUserId(req);
  const { taskId } = params;

  // 2. Get task [P0-B: ownership check]
  const task = await kv.get(taskId);
  if (!task || task.userId !== userId) {
    return Response.json(
      {
        error: "TASK_NOT_FOUND",
        user_message: "任务不存在或已过期",
        code: 404,
      },
      { status: 404 }
    );
  }

  // 3. Check if task is ready
  if (!["completed", "verification_failed"].includes(task.status)) {
    return Response.json(
      {
        error: "TASK_NOT_READY",
        user_message: "任务尚未完成，请稍后查询",
        code: 400,
      },
      { status: 400 }
    );
  }

  // 4. Build response
  const response: Record<string, unknown> = {
    task_id: taskId,
    status: task.status,
  };

  if (task.originalUrl) {
    response.original_url = task.originalUrl;
  }

  if (task.enhancedUrl) {
    response.enhanced_url = task.enhancedUrl;
  }

  if (task.astrometry) {
    response.annotated_info = {
      objects: task.astrometry.objects,
      coordinates: `RA ${task.astrometry.ra}, DEC ${task.astrometry.dec}`,
    };
  }

  if (task.verification) {
    response.verification = task.verification;

    // Add error message for verification failures
    if (!task.verification.authentic) {
      response.error = "VERIFICATION_FAILED";
      response.user_message = "检测到增强图中存在不真实的内容，建议调整参数后重试";
    }
  }

  return Response.json(response);
}
```

---

## Task 15: API路由 — api/verify/[taskId].ts

### 目标
手动触发二次验证。

### 依赖
- **引入包**：`@vercel/kv`
- **引入模块**：`middleware/auth.ts`, `lib/gemini-verify.ts`

### API规格

**端点**：POST /api/verify/:taskId

**Header**：
```
Authorization: Bearer <JWT>
```

**响应（成功）**：
```json
{
  "task_id": "task_abc123",
  "verification": {
    "authentic": true,
    "checks": {
      "star_count": {"pass": true, "detail": "..."},
      "star_position": {"pass": true, "detail": "..."},
      "no_fabricated_objects": {"pass": true, "detail": "..."},
      "no_artifacts": {"pass": true, "detail": "..."},
      "color_authenticity": {"pass": true, "detail": "..."},
      "dynamic_range": {"pass": true, "detail": "..."}
    },
    "verdict": "AUTHENTIC"
  }
}
```

**完整代码**：

```typescript
import { kv } from "@vercel/kv";
import { authMiddleware, getUserId } from "../../middleware/auth";
import { verifyWithGemini } from "../../lib/gemini-verify";

export default async function handler(req: Request, { params }: { params: { taskId: string } }) {
  // 1. Authentication check
  const authError = await authMiddleware(req);
  if (authError) return authError;

  const userId = getUserId(req);
  const { taskId } = params;

  // 2. Get task [P0-B: ownership check]
  const task = await kv.get(taskId);
  if (!task || task.userId !== userId) {
    return Response.json(
      {
        error: "TASK_NOT_FOUND",
        user_message: "任务不存在或已过期",
        code: 404,
      },
      { status: 404 }
    );
  }

  // 3. Check if task has enhanced image
  if (!task.enhancedUrl) {
    return Response.json(
      {
        error: "INVALID_STATUS",
        user_message: "任务尚未完成增强，无法验证",
        code: 400,
      },
      { status: 400 }
    );
  }

  try {
    // 4. Get images
    const [originalResponse, enhancedResponse] = await Promise.all([
      fetch(task.originalUrl),
      fetch(task.enhancedUrl),
    ]);

    const originalBuffer = Buffer.from(await originalResponse.arrayBuffer());
    const enhancedBuffer = Buffer.from(await enhancedResponse.arrayBuffer());

    // 5. Call Gemini verification
    const verification = await verifyWithGemini(
      originalBuffer,
      enhancedBuffer,
      task.astrometry
    );

    // 6. Update task status
    const newStatus = verification.authentic ? "completed" : "verification_failed";
    await kv.set(taskId, {
      ...task,
      status: newStatus,
      verification,
      updatedAt: Date.now(),
    });

    return Response.json({
      task_id: taskId,
      verification,
    });
  } catch (error) {
    console.error("Verification failed:", error);
    return Response.json(
      {
        error: "VERIFICATION_FAILED",
        user_message: "验证过程出错，请稍后重试",
        code: 500,
      },
      { status: 500 }
    );
  }
}
```

---

## Task 16: API路由 — api/quota.ts + api/file/[filename].ts

### 目标
配额查询和文件获取接口。

### 依赖
- **引入包**：`@vercel/kv`, `@vercel/blob`
- **引入模块**：`middleware/auth.ts`, `lib/session-store.ts`

### API规格

#### GET /api/quota

**Header**：
```
Authorization: Bearer <JWT>
```

**响应**：
```json
{
  "user_id": "usr_abc123",
  "plan": "free",
  "daily_limit": 3,
  "used_today": 1,
  "remaining": 2,
  "resets_at": "2026-05-14T00:00:00+08:00"
}
```

**完整代码**：

```typescript
import { authMiddleware, getUserId } from "../middleware/auth";
import { getQuotaInfo } from "../lib/session-store";

export default async function handler(req: Request) {
  // 1. Authentication check
  const authError = await authMiddleware(req);
  if (authError) return authError;

  const userId = getUserId(req);
  if (!userId) {
    return Response.json(
      { error: "UNAUTHORIZED", user_message: "请先登录后使用", code: 401 },
      { status: 401 }
    );
  }

  // 2. Get quota info
  const quota = await getQuotaInfo(userId);

  return Response.json(quota);
}
```

#### GET /api/file/:filename

**Header**：无认证要求（Blob签名URL）

**响应**：直接返回图片文件，或重定向到Blob签名URL

**完整代码**：

```typescript
import { handleRequest } from "@vercel/blob";

export default handleRequest;

// Or for custom handling:
export async function handler(req: Request, { params }: { params: { filename: string } }) {
  const { filename } = params;
  
  // If you need custom access control, implement here
  // Otherwise, delegate to Vercel Blob's built-in handler
  
  // For signed URLs, you can redirect to blob directly:
  // return Response.redirect(blobUrl, 302);
}
```

---

## Task 17: Cron任务 — api/cron/poll-astrometry.ts

### 目标
轮询Astrometry任务状态，使用KV分布式锁防止并发。

### 依赖
- **引入包**：`@vercel/kv`
- **引入模块**：`lib/task-index.ts`, `lib/astrometry.ts`, `lib/constants.ts`

### 配置
**vercel.json**：
```json
{
  "crons": [
    {
      "path": "/api/cron/poll-astrometry",
      "schedule": "* * * * *"
    }
  ]
}
```

**完整代码**：

```typescript
import { kv } from "@vercel/kv";
import { getPendingTasksBatch, removeFromIndex } from "../../lib/task-index";
import { astrometryPollJob, astrometryGetInfo } from "../../lib/astrometry";
import { PENDING_PLATE_TASKS, CRON_LOCK_POLL_ASTROMETRY, PLATE_POLL_BATCH_SIZE } from "../../lib/constants";

export default async function handler() {
  // 1. KV distributed lock [P1-3]
  const lock = await kv.set(CRON_LOCK_POLL_ASTROMETRY, "1", { nx: true, ex: 120 });
  if (!lock) {
    return Response.json({ message: "Previous job still running, skipping" });
  }

  try {
    // 2. Get pending tasks using sscan with timeout protection [P0-E]
    const taskIds = await getPendingTasksBatch(PENDING_PLATE_TASKS, PLATE_POLL_BATCH_SIZE, 5000);

    if (taskIds.length === 0) {
      return Response.json({ processed: 0, message: "No pending tasks" });
    }

    // 3. Process in batches
    const results = [];
    const batchSize = PLATE_POLL_BATCH_SIZE;

    for (let i = 0; i < taskIds.length; i += batchSize) {
      const batch = taskIds.slice(i, i + batchSize);
      const batchResults = await Promise.allSettled(
        batch.map(async (taskId) => {
          const task = await kv.get(taskId);
          if (!task) {
            await removeFromIndex(PENDING_PLATE_TASKS, taskId);
            return { taskId, status: "skipped", reason: "task_not_found" };
          }

          // Timeout check (5 minutes)
          if (Date.now() - task.createdAt > 300_000) {
            await kv.set(taskId, { ...task, status: "plate_failed", error: "timeout" });
            await removeFromIndex(PENDING_PLATE_TASKS, taskId);
            return { taskId, status: "timeout" };
          }

          // Poll Astrometry
          const result = await astrometryPollJob(task.subid);

          if (result.status === "success") {
            const info = await astrometryGetInfo(result.jobId!);
            await kv.set(taskId, {
              ...task,
              status: "plate_solved",
              jobId: result.jobId,
              astrometry: {
                ra: info.ra,
                dec: info.dec,
                objects: info.objects,
                pixscale: info.pixscale,
                radius: info.radius,
                orientation: info.orientation,
              },
              updatedAt: Date.now(),
            });
            await removeFromIndex(PENDING_PLATE_TASKS, taskId);
            return { taskId, status: "success" };
          } else if (result.status === "failure") {
            await kv.set(taskId, { ...task, status: "plate_failed", error: "solve_failed", updatedAt: Date.now() });
            await removeFromIndex(PENDING_PLATE_TASKS, taskId);
            return { taskId, status: "failed" };
          } else {
            // Still processing, keep in index
            return { taskId, status: "processing" };
          }
        })
      );

      results.push(...batchResults);
    }

    // 4. Summarize results
    const summary = {
      processed: taskIds.length,
      success: results.filter(r => r.status === "fulfilled" && r.value.status === "success").length,
      failed: results.filter(r => r.status === "fulfilled" && r.value.status === "failed").length,
      timeout: results.filter(r => r.status === "fulfilled" && r.value.status === "timeout").length,
      processing: results.filter(r => r.status === "fulfilled" && r.value.status === "processing").length,
    };

    return Response.json(summary);
  } finally {
    // 5. Release lock
    await kv.del(CRON_LOCK_POLL_ASTROMETRY);
  }
}
```

---

## Task 18: Cron任务 — api/cron/process-enhance.ts

### 目标
处理增强队列中的任务，使用KV分布式锁。

### 依赖
- **引入包**：`@vercel/kv`
- **引入模块**：`lib/task-index.ts`, `lib/gemini-enhance.ts`, `lib/prompt-builder.ts`, `lib/file-store.ts`, `lib/constants.ts`

### 配置
**vercel.json**：
```json
{
  "crons": [
    {
      "path": "/api/cron/process-enhance",
      "schedule": "*/5 * * * *"
    }
  ]
}
```

**完整代码**：

```typescript
import { kv } from "@vercel/kv";
import { getPendingTasksBatch, removeFromIndex } from "../../lib/task-index";
import { enhanceWithGemini, enhanceWithGeminiAndVerify } from "../../lib/gemini-enhance";
import { buildEnhancementPrompt } from "../../lib/prompt-builder";
import { uploadEnhancedImage } from "../../lib/file-store";
import { PENDING_ENHANCE_TASKS, CRON_LOCK_PROCESS_ENHANCE, ENHANCE_POLL_BATCH_SIZE } from "../../lib/constants";

export default async function handler() {
  // 1. KV distributed lock [P1-3]
  const lock = await kv.set(CRON_LOCK_PROCESS_ENHANCE, "1", { nx: true, ex: 300 });
  if (!lock) {
    return Response.json({ message: "Previous job still running, skipping" });
  }

  try {
    // 2. Get pending enhance tasks
    const taskIds = await getPendingTasksBatch(PENDING_ENHANCE_TASKS, ENHANCE_POLL_BATCH_SIZE, 10000);

    if (taskIds.length === 0) {
      return Response.json({ processed: 0 });
    }

    // 3. Process in batches
    const batchSize = ENHANCE_POLL_BATCH_SIZE;
    for (let i = 0; i < taskIds.length; i += batchSize) {
      const batch = taskIds.slice(i, i + batchSize);
      await Promise.allSettled(
        batch.map(async (taskId) => {
          const task = await kv.get(taskId);
          if (!task) {
            await removeFromIndex(PENDING_ENHANCE_TASKS, taskId);
            return { taskId, status: "skipped" };
          }

          try {
            // Get original image
            const imageResponse = await fetch(task.originalUrl);
            const imageBuffer = Buffer.from(await imageResponse.arrayBuffer());

            // Build prompt
            const prompt = buildEnhancementPrompt(task.astrometry || null, task.style || "natural");

            // Call Gemini
            const hasVerification = task.status === "verifying";
            const result = hasVerification
              ? await enhanceWithGeminiAndVerify(imageBuffer, task.astrometry || null, task.style || "natural")
              : await enhanceWithGemini(imageBuffer, task.astrometry || null, task.style || "natural");

            // Upload enhanced image
            const blob = await uploadEnhancedImage(taskId, result.images[0]);
            
            // Update task
            await kv.set(taskId, {
              ...task,
              status: hasVerification ? "verifying" : "completed",
              enhancedUrl: blob.url,
              verification: result.verification,
              updatedAt: Date.now(),
            });
          } catch (error) {
            console.error(`Failed to process task ${taskId}:`, error);
            await kv.set(taskId, {
              ...task,
              status: "plate_solved", // Revert to previous status
              error: "GEMINI_ENHANCE_FAILED",
            });
          }

          // Remove from queue
          await removeFromIndex(PENDING_ENHANCE_TASKS, taskId);
        })
      );
    }

    return Response.json({ processed: taskIds.length });
  } finally {
    // 4. Release lock
    await kv.del(CRON_LOCK_PROCESS_ENHANCE);
  }
}
```

---

## Task 19: 配置文件 — vercel.json + package.json + tsconfig.json + .env.example

### 目标
提供完整的项目配置文件。

### 依赖
- **引入包**：见package.json
- **引入模块**：无

### vercel.json

```json
{
  "functions": {
    "api/cron/poll-astrometry.ts": {
      "maxDuration": 60
    },
    "api/cron/process-enhance.ts": {
      "maxDuration": 60
    },
    "api/cron/process-overlay.ts": {
      "maxDuration": 60
    },
    "api/enhance/[taskId].ts": {
      "maxDuration": 60
    },
    "api/verify/[taskId].ts": {
      "maxDuration": 60
    }
  },
  "crons": [
    {
      "path": "/api/cron/poll-astrometry",
      "schedule": "* * * * *"
    },
    {
      "path": "/api/cron/process-enhance",
      "schedule": "*/5 * * * *"
    },
    {
      "path": "/api/cron/process-overlay",
      "schedule": "*/5 * * * *"
    }
  ]
}
```

### package.json

```json
{
  "name": "astrorender-backend",
  "version": "2.1.0",
  "description": "AstroRender - Astronomical Photo Enhancement Service",
  "main": "api/index.ts",
  "scripts": {
    "dev": "vercel dev",
    "build": "tsc",
    "deploy": "vercel --prod",
    "test": "vitest"
  },
  "dependencies": {
    "@google/generative-ai": "^0.21.0",
    "@vercel/blob": "^0.22.0",
    "@vercel/kv": "^1.0.0",
    "hono": "^4.0.0",
    "jose": "^5.2.0",
    "sharp": "^0.33.0"   // [v2.2-added] 图像处理库
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.3.0",
    "vercel": "^33.0.0",
    "vitest": "^1.0.0"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022"],
    "types": ["@vercel/node"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "allowJs": true,
    "noEmit": true,
    "isolatedModules": true
  },
  "include": ["api/**/*", "lib/**/*", "middleware/**/*"],
  "exclude": ["node_modules", "__tests__"]
}
```

### .env.example

```bash
# Astrometry.net
ASTROMETRY_API_KEY=your_astrometry_api_key

# Gemini
GEMINI_API_KEY=your_gemini_api_key

# JWT Authentication [P0-C: No default value]
JWT_SECRET=your-32-character-secret-key-here

# Vercel KV (auto-injected by Vercel, no manual config needed)
# KV_REST_API_URL=
# KV_REST_API_TOKEN=

# Vercel Blob (auto-injected by Vercel, no manual config needed)
# BLOB_READ_WRITE_TOKEN=

# Optional Configuration
MAX_FILE_SIZE=20971520        # 20MB
ASTROMETRY_TIMEOUT=300        # 5 minutes
MAX_FREE_QUOTA_PER_USER=3      # Free user daily limit
MAX_PRO_QUOTA_PER_USER=30      # Pro user daily limit
GEMINI_BATCH_SIZE=5           # Cron parallel processing batch size

# Constellation overlay [v2.2-added]
CONSTELLATION_MATCH_THRESHOLD=0.6
```

---

## Task 20: 星座数据库 — lib/constellation-data.ts

### 目标
定义16个星座的完整数据：主星坐标、连线拓扑、神话描述、边界范围。

### 依赖
- **引入包**：无
- **引入模块**：`lib/types.ts`
- **使用类型**：`ConstellationDef`, `ConstellationStar`

### 数据要求

16个星座，每个星座需要：

1. **主星列表**：使用HIPPARCOS星表数据，包含星名、赤经(度)、赤纬(度)、视星等、是否主星
2. **连线拓扑**：用星点索引对表示，如 `[[0,1], [1,2]]`
3. **预计算边界**：ra_min, ra_max, dec_min, dec_max

具体星座连线拓扑参考（经典连线方式）：

**Aries (白羊座)** - 4主星：Hamal, Sheratan, Mesarthim, 41 Ari
连线：Hamal-Sheratan, Sheratan-Mesarthim, Mesarthim-41 Ari

**Taurus (金牛座)** - 7主星：Aldebaran, Elnath, Alcyone(昴宿星团代表), Ain, Tianguan, Prima Hyadum, Secunda Hyadum
连线：Aldebaran-Prima Hyadum, Prima Hyadum-Secunda Hyadum, Aldebaran-Ain, Aldebaran-Elnath, Aldebaran-Tianguan, Alcyone(独立标注)

**Gemini (双子座)** - 6主星：Castor, Pollux, Alhena, Tejat, Mebsuta, Wasat
连线：Castor-Pollux, Castor-Mebsuta, Pollux-Alhena, Mebsuta-Wasat, Wasat-Alhena

**Cancer (巨蟹座)** - 5主星：Acubens, Al Tarf, Asellus Borealis, Asellus Australis, Iota Cancri
连线：Acubens-Asellus Borealis, Asellus Borealis-Asellus Australis, Asellus Australis-Al Tarf, Asellus Borealis-Iota Cancri

**Leo (狮子座)** - 9主星：Regulus, Denebola, Algieba, Zosma, Ras Elased, Chertan, Adhafera, Eta Leonis, Mu Leonis
连线：Regulus-Eta Leonis, Eta Leonis-Algieba, Algieba-Adhafera, Algieba-Ras Elased, Eta Leonis-Mu Leonis, Adhafera-Chertan, Chertan-Zosma, Zosma-Denebola

**Virgo (室女座)** - 7主星：Spica, Zavijava, Porrima, Auva, Vindemiatrix, Heze, Minelauva
连线：Spica-Zavijava, Zavijava-Porrima, Porrima-Auva, Auva-Minelauva, Porrima-Vindemiatrix, Vindemiatrix-Heze

**Libra (天秤座)** - 4主星：Zubeneschamali, Zubenelgenubi, Brachium, Sigma Librae
连线：Zubeneschamali-Zubenelgenubi, Zubeneschamali-Brachium, Brachium-Sigma Librae, Zubenelgenubi-Sigma Librae

**Scorpius (天蝎座)** - 8主星：Antares, Shaula, Sargas, Dschubba, Graffias, Wei, Sargas, Lesath
连线：Graffias-Dschubba, Dschubba-Antares, Antares-Wei, Wei-Shaula, Shaula-Lesath, Antares-Sargas

**Sagittarius (射手座)** - 8主星：Kaus Australis, Nunki, Ascella, Kaus Media, Kaus Borealis, Alnasl, Phikda, Rukbat
连线：Kaus Borealis-Kaus Media, Kaus Media-Kaus Australis, Kaus Media-Alnasl, Kaus Australis-Ascella, Ascella-Nunki, Phikda-Kaus Australis, Rukbat-Phikda

**Capricornus (摩羯座)** - 5主星：Deneb Algedi, Dabih, Algedi, Nashira, Iota Capricorni
连线：Algedi-Dabih, Dabih-Nashira, Nashira-Deneb Algedi, Dabih-Iota Capricorni

**Aquarius (水瓶座)** - 7主星：Sadalsuud, Sadalmelik, Skat, Albali, Ancha, Eta Aquarii, Lambda Aquarii
连线：Sadalsuud-Sadalmelik, Sadalmelik-Eta Aquarii, Eta Aquarii-Ancha, Sadalsuud-Skat, Skat-Albali, Eta Aquarii-Lambda Aquarii

**Pisces (双鱼座)** - 7主星：Eta Piscium, Alrescha, Fumalsamakah, Delta Piscium, Epsilon Piscium, Omega Piscium, Iota Piscium
连线：Eta Piscium-Alrescha, Alrescha-Fumalsamakah, Fumalsamakah-Delta Piscium, Delta Piscium-Epsilon Piscium, Alrescha-Iota Piscium, Epsilon Piscium-Omega Piscium

**Orion (猎户座)** - 7主星：Betelgeuse, Rigel, Bellatrix, Saiph, Mintaka, Alnilam, Alnitak
连线：Betelgeuse-Bellatrix, Bellatrix-Mintaka, Mintaka-Alnilam, Alnilam-Alnitak, Alnitak-Saiph, Saiph-Rigel, Rigel-Mintaka

**Lyra (天琴座)** - 4主星：Vega, Sheliak, Sulafat, Delta2 Lyrae
连线：Vega-Sheliak, Vega-Sulafat, Sheliak-Sulafat, Vega-Delta2 Lyrae

**Ursa Major (大熊座)** - 7主星：Dubhe, Merak, Phecda, Megrez, Alioth, Mizar, Alkaid
连线：Dubhe-Merak, Merak-Phecda, Phecda-Megrez, Megrez-Alioth, Alioth-Mizar, Mizar-Alkaid (北斗七星经典连线)

**Ursa Minor (小熊座)** - 4主星：Polaris, Kochab, Pherkad, Eta Ursae Minoris
连线：Polaris-Kochab, Kochab-Pherkad, Pherkad-Eta Ursae Minoris, Eta Ursae Minoris-Polaris

### 导出

```typescript
export const CONSTELLATIONS: ConstellationDef[] = [ /* 16个星座数据 */ ];

// 按名称快速查找
export function getConstellationByName(name: string): ConstellationDef | undefined;

// 按坐标范围查找可能匹配的星座
export function findCandidateConstellations(ra: number, dec: number, radius: number): ConstellationDef[];
```

### 星点坐标来源
使用 HIPPARCOS 星表数据，赤经赤纬转换为度数（J2000历元）。开发者需从 IAU 官方星座边界数据或天文数据库获取精确坐标。

---

## Task 21: 星座识别匹配 — lib/constellation-match.ts

### 目标
根据 Astrometry.net 返回的 plate-solving 结果，匹配用户照片中包含的星座。

### 依赖
- **引入包**：无
- **引入模块**：`lib/types.ts`, `lib/constellation-data.ts`, `lib/constants.ts`
- **使用类型**：`AstrometryData`, `DetectedConstellation`, `ConstellationDef`

### 函数签名

```typescript
/**
 * 根据plate-solving结果，识别照片中包含的星座
 * @param astrometry - Astrometry返回的天文数据
 * @returns 检测到的星座列表
 */
export function detectConstellations(
  astrometry: AstrometryData
): DetectedConstellation[];

/**
 * 将Astrometry返回的天体名称与星座库中的星点做匹配
 * @param objectsInField - Astrometry返回的objects_in_field
 * @param candidateConstellations - 候选星座列表
 * @returns 每个候选星座的匹配率
 */
export function matchStarsToConstellations(
  objectsInField: string[],
  candidateConstellations: ConstellationDef[]
): Map<string, number>;
```

### 匹配算法
1. 从 Astrometry 的 `objects_in_field` 和 WCS 坐标信息提取照片的 FOV 和中心坐标
2. 调用 `findCandidateConstellations` 获取边界与FOV重叠的候选星座
3. 对每个候选星座：
   a. 将星座主星坐标通过WCS逆变换映射到图像像素坐标
   b. 检查有多少主星落在图像范围内
   c. 计算匹配率 = 落在图像内的主星数 / 总主星数
   d. 如果匹配率 >= `CONSTELLATION_MATCH_THRESHOLD`，判定为检测到
4. 返回所有匹配率超过阈值的星座

### 安全要求
- P1: 坐标转换需处理赤经环绕（0°/360°边界）的特殊情况
- P2: 匹配结果缓存到KV，避免重复计算

---

## Task 22: 星座叠加合成 — lib/constellation-overlay.ts

### 目标
实现分层叠加合成逻辑：在用户增强图上叠加星座连线、神话图案和装饰元素。

### 依赖
- **引入包**：`sharp`（图像处理）
- **引入模块**：`lib/types.ts`, `lib/constellation-data.ts`, `lib/constants.ts`, `lib/file-store.ts`
- **使用类型**：`Task`, `ConstellationDef`, `ConstellationStyle`, `DetectedConstellation`

### 函数签名

```typescript
/**
 * 执行星座叠加合成 [v2.2-revised]
 * @param enhancedImagePath - 用户增强图路径(Vercel Blob URL)
 * @param constellation - 星座定义
 * @param astrometry - Astrometry校准数据(用于WCS坐标转换)
 * @returns 叠加后图片的Blob URL
 */
export async function overlayConstellation(
  enhancedImagePath: string,
  constellation: ConstellationDef,
  astrometry: AstrometryData
): Promise<string>;

/**
 * 根据WCS信息将赤经赤纬转换为图像像素坐标
 * @param ra - 赤经(度)
 * @param dec - 赤纬(度)
 * @param astrometry - WCS校准数据
 * @param imageWidth - 图像宽度
 * @param imageHeight - 图像高度
 * @returns 像素坐标 {x, y}
 */
export function wcsToPixel(
  ra: number,
  dec: number,
  astrometry: AstrometryData,
  imageWidth: number,
  imageHeight: number
): { x: number; y: number };

/**
 * 计算神话图案PNG的仿射变换参数
 * @param constellation - 星座定义
 * @param starPixelPositions - 星座主星在图像上的像素坐标
 * @returns 仿射变换参数 {scale, rotation, translateX, translateY}
 */
export function computeMythologyTransform(
  constellation: ConstellationDef,
  starPixelPositions: Map<number, {x: number; y: number}>
): { scale: number; rotation: number; translateX: number; translateY: number };

/**
 * 在图像上绘制星座连线
 * @param sharpImage - Sharp图像实例
 * @param starPixelPositions - 星座主星像素坐标
 * @param lines - 连线索引对
 * @param styleConfig - 风格配置
 * @returns SVG overlay字符串
 */
export function drawConstellationLines(
  starPixelPositions: Map<number, {x: number; y: number}>,
  lines: [number, number][],
  styleConfig: StyleConfig
): string;

/**
 * 在图像上绘制装饰元素(网格、边框、标题) [v2.2-revised]
 * @param imageWidth - 图像宽度
 * @param imageHeight - 图像高度
 * @param constellation - 星座定义
 * @returns SVG overlay字符串
 */
export function drawDecorations(
  imageWidth: number,
  imageHeight: number,
  constellation: ConstellationDef
): string;
```

### 实现要点

1. **WCS坐标转换**：
   - Astrometry返回的calibration数据包含CD矩阵(或CDELT1/CDELT2 + CROTA2)
   - 使用标准WCS逆变换公式将(RA,DEC)转换为像素坐标
   - 需处理图像方向(orientation)和像素比例尺(pixscale)

2. **连线绘制**：
   - 使用SVG overlay方式（Sharp支持SVG composite）
   - 星点：带高斯模糊光晕的圆点
   - 连线：指定颜色、宽度、透明度的线段

3. **神话图案叠加**：
   - 加载对应的预置PNG（`assets/mythology/{style}/{name}.png`）
   - 根据星座关键星点（如头部星点、脚部星点）计算仿射变换
   - 使用Sharp的composite功能叠加，设置混合模式

4. **装饰绘制**：
   - 坐标网格：根据FOV绘制等间距赤经赤纬线
   - 边框：根据风格配置绘制装饰边框
   - 标题：根据风格配置绘制文字（baroque用中文+英文，19th_century仅英文）

5. **输出**：
   - 最终合成图保存到Vercel Blob
   - 命名格式：`{task_id}_overlay_{constellation}_{style}.png`

### 安全要求
- P0: 图片处理需设置超时(30s)，避免大图卡住
- P1: SVG模板需防注入（星座名和标题需转义）
- P1: 神话图案PNG需校验尺寸，防止异常大图

### 依赖新增
package.json 新增：`sharp` (图像处理库)

---

## Task 23: API路由 — api/constellation-overlay/[taskId].ts

### 目标
星座叠加请求的API入口。

### 依赖
- **引入包**：`@vercel/kv`
- **引入模块**：`lib/types.ts`, `lib/auth.ts`, `lib/session-store.ts`, `lib/constellation-match.ts`, `lib/constellation-overlay.ts`, `lib/constellation-data.ts`, `lib/task-index.ts`
- **使用类型**：`Task`, `DetectedConstellation`, `ConstellationStyle`

### 路由定义 [v2.2-revised]

```
POST /api/constellation-overlay/:task_id
Authorization: Bearer <JWT>
Content-Type: application/json

Request Body:
{
  "constellation": "orion"
}
```

### 处理流程 [v2.2-revised]

1. **认证**：验证JWT，获取userId
2. **校验taskId**：从KV获取任务，验证userId归属（P0安全）
3. **校验状态**：任务必须处于 `completed` 状态
4. **配额检查**：检查用户当日配额剩余，不足返回 QUOTA_EXCEEDED
5. **校验参数**：
   - constellation：必须存在于星座库中
6. **星座验证**：调用 `detectConstellations` 确认该星座确实在照片中被识别到
7. **更新状态**：KV写入 status="overlaying"
8. **异步处理**：将任务加入 `pending_overlay_tasks` 索引集合
9. **返回响应**

### 响应格式 [v2.2-revised]

成功：
```json
{
  "success": true,
  "task_id": "task_abc123",
  "status": "overlaying",
  "constellation": "orion",
  "style": "vintage",
  "message": "正在生成星座叠加图",
  "estimated_wait_seconds": 15
}
```

错误：
```json
{
  "success": false,
  "error": "CONSTELLATION_NOT_DETECTED",
  "user_message": "未在照片中识别到支持的星座",
  "code": 404
}
```

### 错误码 [v2.2-revised]
| 条件 | 错误码 | HTTP状态 | user_message |
|------|--------|----------|-------------|
| 星座不在库中 | INVALID_CONSTELLATION | 400 | 不支持的星座名称 |
| 任务未完成 | INVALID_STATUS_TRANSITION | 400 | 当前状态不支持该操作 |
| 星座未识别 | CONSTELLATION_NOT_DETECTED | 404 | 未在照片中识别到支持的星座 |
| 配额不足 | QUOTA_EXCEEDED | 429 | 今日使用次数已用完 |
// [v2.2-revised] INVALID_OVERLAY_STYLE 已移除，风格已固定为 vintage

### 安全要求
- P0-B: taskId归属校验（与现有接口一致）
- P1: constellation参数需白名单校验，防止路径遍历

---

## Task 24: Cron任务 — api/cron/process-overlay.ts [v2.2-revised]

### 目标
定时处理星座叠加任务队列，包含二次验证步骤。

### 依赖
- **引入包**：`@vercel/kv`
- **引入模块**：`lib/types.ts`, `lib/constellation-overlay.ts`, `lib/overlay-verify.ts`, `lib/constellation-data.ts`, `lib/task-index.ts`, `lib/file-store.ts`, `lib/session-store.ts`
- **使用类型**：`Task`, `OverlayVerificationResult`

### 处理流程 [v2.2-revised]

```
Vercel Cron (每5分钟):
  1. 获取分布式锁(KV SETNX, TTL=4min)
  2. sscan从 pending_overlay_tasks 获取所有待处理任务ID
  3. 批量并行处理每个任务(Promise.allSettled):
     a. 从KV读取任务详情
     b. 获取增强图的Blob URL
     c. 获取星座定义数据
     d. 调用 overlayConstellation() 执行叠加合成
     e. 更新KV: status="overlay_verifying" [v2.2-revised]
     f. 调用 verifyOverlay() 执行二次验证 [v2.2-revised]
     g. 如果验证通过: status="overlay_passed" → "completed", 保存overlay_url [v2.2-revised]
     h. 如果验证未通过: status="overlay_failed", 保留原有增强结果 [v2.2-revised]
     i. 从索引集合移除task_id
  4. 释放分布式锁
```

### 验证通过流程 [v2.2-revised]
```
overlayConstellation() → overlay_verifying → verifyOverlay()
                                              ↓
                              ┌───────────────┴───────────────┐
                              ↓                               ↓
                      overlay_passed                    overlay_failed
                      (验证通过)                       (验证未通过)
                              ↓                               ↓
                         completed                       completed
                    (带overlay_url)              (无overlay, 保留原增强结果)
```

### 超时配置
- 单个叠加任务超时：30秒
- 验证任务超时：30秒
- Cron总执行时间限制：60秒（Vercel Pro）
- 并发处理数量：GEMINI_BATCH_SIZE（默认5）

### 安全要求
- P1-3: KV分布式锁防并发（与现有Cron一致）
- P0-E: sscan分批迭代+超时保护
- P1: 单任务超时保护，避免一个卡住影响整批

---

## Task 25: 星座叠加二次验证 — lib/overlay-verify.ts [v2.2-revised]

### 目标
对星座叠加结果进行二次验证，确保叠加内容真实，没有添加伪造天体。

### 依赖
- **引入包**：`@google/generative-ai`
- **引入模块**：`lib/types.ts`, `lib/constellation-data.ts`, `lib/constants.ts`
- **使用类型**：`OverlayVerificationResult`, `DetectedConstellation`, `AstrometryData`

### 函数签名

```typescript
import { GoogleGenerativeAI } from "@google/generative-ai";
import { GEMINI_MODEL_VERIFY, GEMINI_VERIFY_TIMEOUT } from "./constants";
import type { OverlayVerificationResult, DetectedConstellation, AstrometryData } from "./types";

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);

/**
 * 星座叠加验证Prompt [v2.2-revised]
 */
export const OVERLAY_VERIFY_PROMPT = `
You are a scientific verification expert for astronomical image overlays.

TASK: Verify that the constellation overlay in this image is AUTHENTIC and does NOT contain fabricated celestial objects.

Original image context:
- Detected constellation: {constellation_name} ({constellation_zh})
- Center coordinates: RA {ra}, DEC {dec}
- Field of view radius: {radius}°
- Identified objects: {objects}

Overlay verification criteria:

1. CONSTELLATION_MATCH: Does the overlaid figure match the detected constellation ({constellation_name})?
   - Check if the mythological figure/pattern corresponds to the correct constellation
   - Check if the Chinese title "{constellation_zh}" and English title "{constellation_en}" are present and correct

2. STAR_ALIGNMENT: Are the constellation lines and star positions accurately placed?
   - Stars should be connected according to the standard constellation pattern
   - Star positions should be approximately correct relative to the overlay figure

3. NO_FABRICATED_OBJECTS: Are there any additional celestial objects that were NOT in the original image?
   - Extra stars, nebulae, or galaxies that don't exist in the original exposure
   - Any mythological elements that contradict the constellation's traditional depiction

4. STYLE_CONSISTENCY: Is the overlay style consistent and free from obvious artifacts?
   - The overlay should appear as a unified vintage astronomical chart
   - No obvious seams, color mismatches, or compositing artifacts

Respond in JSON format:
{
  "constellation_match": { "pass": true/false, "detail": "..." },
  "star_alignment": { "pass": true/false, "detail": "..." },
  "no_fabricated_objects": { "pass": true/false, "detail": "..." },
  "style_consistency": { "pass": true/false, "detail": "..." },
  "verdict": "PASS" if all checks pass, "FAIL" otherwise,
  "issues": ["list of specific issues if any"]
}
`.trim();

/**
 * 对星座叠加结果进行二次验证 [v2.2-revised]
 * @param overlayImagePath - 叠加后的图片路径(Vercel Blob URL)
 * @param originalImagePath - 用户原始增强图路径(Vercel Blob URL)
 * @param detectedConstellation - 检测到的星座信息
 * @param astrometry - Astrometry校准数据
 * @returns 验证结果
 */
export async function verifyOverlay(
  overlayImagePath: string,
  originalImagePath: string,
  detectedConstellation: DetectedConstellation,
  astrometry: AstrometryData
): Promise<OverlayVerificationResult>;
```

### 验证检查项 [v2.2-revised]

1. **CONSTELLATION_MATCH**：叠加图案是否与检测到的星座匹配
   - 检查神话图案是否对应正确的星座
   - 检查中英文标题是否正确

2. **STAR_ALIGNMENT**：星座连线和星点位置是否准确放置
   - 星点应按标准星座图案连接
   - 星点位置应与叠加图案相对正确

3. **NO_FABRICATED_OBJECTS**：是否有额外添加的不存在天体
   - 检查原始图像中没有的额外星点、星云或星系
   - 检查是否有与星座传统描绘矛盾的神话元素

4. **STYLE_CONSISTENCY**：叠加风格是否一致，无明显瑕疵
   - 叠加应呈现为统一的复古星图风格
   - 无明显的接缝、颜色不匹配或合成痕迹

### 实现要点

```typescript
/**
 * 验证星座叠加结果的真实性
 */
export async function verifyOverlay(
  overlayImagePath: string,
  originalImagePath: string,
  detectedConstellation: DetectedConstellation,
  astrometry: AstrometryData
): Promise<OverlayVerificationResult> {
  const model = genAI.getGenerativeModel({
    model: GEMINI_MODEL_VERIFY,
    generationConfig: {
      responseModalities: ["TEXT"],
    },
  });

  // 获取图片
  const [originalBuffer, overlayBuffer] = await Promise.all([
    fetch(originalImagePath).then(r => r.arrayBuffer()),
    fetch(overlayImagePath).then(r => r.arrayBuffer()),
  ]);

  const originalBase64 = Buffer.from(originalBuffer).toString("base64");
  const overlayBase64 = Buffer.from(overlayBuffer).toString("base64");

  // 构建验证Prompt
  const prompt = buildOverlayVerifyPrompt(detectedConstellation, astrometry);

  // 调用Gemini验证
  const result = await Promise.race([
    model.generateContent([
      { text: prompt },
      { inlineData: { mimeType: "image/jpeg", data: originalBase64 } },
      { inlineData: { mimeType: "image/png", data: overlayBase64 } },
    ]),
    new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error("Verify timeout")), GEMINI_VERIFY_TIMEOUT * 1000)
    ),
  ]);

  // 解析响应
  const text = result.response.text();
  const jsonMatch = text.match(/\{[\s\S]*\}/);
  
  if (jsonMatch) {
    const parsed = JSON.parse(jsonMatch[0]);
    return {
      authentic: parsed.verdict === "PASS",
      checks: {
        constellation_match: { pass: parsed.constellation_match?.pass, detail: parsed.constellation_match?.detail || "" },
        star_alignment: { pass: parsed.star_alignment?.pass, detail: parsed.star_alignment?.detail || "" },
        no_fabricated_objects: { pass: parsed.no_fabricated_objects?.pass, detail: parsed.no_fabricated_objects?.detail || "" },
        style_consistency: { pass: parsed.style_consistency?.pass, detail: parsed.style_consistency?.detail || "" },
      },
      issues: parsed.issues || [],
      verdict: parsed.verdict,
    };
  }

  // 默认返回失败
  return {
    authentic: false,
    checks: {
      constellation_match: { pass: false, detail: "验证解析失败" },
      star_alignment: { pass: false, detail: "验证解析失败" },
      no_fabricated_objects: { pass: false, detail: "验证解析失败" },
      style_consistency: { pass: false, detail: "验证解析失败" },
    },
    issues: ["验证结果解析失败"],
    verdict: "FAIL",
  };
}
```

### 安全要求
- P1: 验证超时设置为30秒，避免长时间等待
- P1: 解析JSON时处理异常情况
- P2: 验证结果记录到KV，便于问题排查

---

## 附录：状态机

```
                    ┌───────────┐
                    │  uploaded  │ ← 初始状态
                    └─────┬─────┘
                          │ 触发plate-solving
                          ▼
                    ┌──────────────┐
                    │ plate_solving │ ← Astrometry处理中
                    └─────┬────────┘
                          │
              ┌───────────┼───────────┐
              ▼                       ▼
      ┌──────────────┐       ┌──────────────┐
      │ plate_solved │       │ plate_failed │
      └──────┬───────┘       └──────┬───────┘
             │                      │
             │ 用户触发增强          │ 降级：跳过WCS
             ▼                      ▼
      ┌──────────────┐       ┌──────────────┐
      │  enhancing   │◄──────│  enhancing   │
      └──────┬───────┘  (无WCS) └──────┬───────┘
             │                         │
             ▼                         ▼
      ┌───────────────────────────────────┐
      │           verifying               │ ← 二次验证（可跳过）
      └───────────────┬───────────────────┘
                      │
          ┌───────────┼───────────┐
          ▼                       ▼
  ┌──────────────┐       ┌──────────────────┐
  │  completed   │       │verification_failed│
  └──────┬───────┘       └────────┬─────────┘
         │                        │ 可重试增强
         │                        └──→ enhancing
         │
         │ 用户触发星座叠加
         ▼
  ┌──────────────┐
  │  overlaying  │ ← [v2.2-added] 星座叠加处理中
  └──────┬───────┘
         │
         ▼
  ┌───────────────────┐
  │overlay_verifying │ ← [v2.2-revised] 星座叠加图二次验证中
  └──────┬────────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌────────┐ ┌──────────────┐
│overlay_ │ │overlay_failed│ ← [v2.2-revised] 叠加失败或验证未通过
│passed  │ └──────────────┘
│(验证通过)│
└────┬───┘
     ▼
┌────────┐
│completed│
│(带overlay)│
└────────┘
```

星座叠加分支说明 [v2.2-added] [v2.2-revised]:

```
completed ──→ overlaying ──→ overlay_verifying ──→ completed (带overlay_url)
                                     │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
              overlay_passed    overlay_failed     completed(无overlay)
              (验证通过)        (验证未通过)       (保留原增强结果)
```

新增状态说明：
- `overlay_verifying` [v2.2-revised]：星座叠加图二次验证中
- `overlay_passed` [v2.2-revised]：叠加验证通过（临时状态）
- `overlay_failed` [v2.2-revised]：叠加失败或验证未通过，不影响已有的增强结果

星座叠加消耗1次配额

---

## 附录：错误码对照表 [v2.2-revised]

| 错误码 | HTTP状态 | user_message | 说明 |
|--------|----------|--------------|------|
| FILE_TOO_LARGE | 400 | 图片文件过大，请上传20MB以下的图片 | 文件超过限制 |
| UNSUPPORTED_FORMAT | 400 | 暂不支持该图片格式，请上传JPG/PNG/FITS格式 | 文件格式不支持 |
| INVALID_INPUT | 400 | 输入参数错误 | 参数校验失败 |
| INVALID_STATUS_TRANSITION | 400 | 当前状态不支持该操作 | 状态机错误 |
| INVALID_CONSTELLATION | 400 | 不支持的星座名称 | 星座名不在库中 [v2.2-added] |
// [v2.2-revised] INVALID_OVERLAY_STYLE 已移除，风格已固定为 vintage
| UNAUTHORIZED | 401 | 请先登录后使用 | 未认证 |
| INVALID_TOKEN | 401 | 登录已过期，请重新登录 | Token无效 |
| INVALID_CREDENTIALS | 401 | 邮箱或密码错误 | 登录失败 |
| TASK_NOT_FOUND | 404 | 任务不存在或已过期 | 任务不存在或无权限 |
| CONSTELLATION_NOT_DETECTED | 404 | 未在照片中识别到支持的星座 | 星座匹配率低于阈值 [v2.2-added] |
| QUOTA_EXCEEDED | 429 | 今日使用次数已用完 | 配额超限 |
| ASTROMETRY_LOGIN_FAILED | 500 | 星图识别服务暂时不可用，请稍后重试 | Astrometry登录失败 |
| ASTROMETRY_UPLOAD_FAILED | 500 | 图片上传失败，请检查网络后重试 | Astrometry上传失败 |
| GEMINI_ENHANCE_FAILED | 504 | 图像增强失败，请稍后重试 | Gemini调用失败 |
| VERIFICATION_FAILED | 500 | 检测到增强图中存在不真实的内容，建议调整参数后重试 | 验证未通过 |
| OVERLAY_VERIFICATION_FAILED [v2.2-revised] | 500 | 星座叠加图验证未通过，请稍后重试 | 叠加图不符合真实性要求 |
