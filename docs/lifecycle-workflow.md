# Typing Test Lifecycle Workflow

This document traces the complete lifecycle of a typing test session in Monkeytype, from initial configuration to saving results in the database.

## Table of Contents

- [1. Session Initiation](#1-session-initiation)
- [2. During Test](#2-during-test)
- [3. Test Completion](#3-test-completion)
- [4. Result Submission](#4-result-submission)
- [5. Backend Processing](#5-backend-processing)
- [Flow Diagram](#flow-diagram)
- [Anti-Cheat Measures](#anti-cheat-measures)
- [File References](#file-references)

---

## 1. Session Initiation

### Configuration Setup

**File:** `frontend/src/ts/test/test-config.ts`

The test page initializes with a configuration UI that allows users to select:

- **Mode**: time, words, quote, custom, zen
- **Duration/Count**:
  - Time mode: 15, 30, 60, 120 seconds
  - Words mode: 10, 25, 50, 100 words
  - Quote mode: short, medium, long, thicc
- **Options**: punctuation, numbers, language, difficulty
- **Other settings**: lazyMode, blindMode, stopOnError, funboxes

```typescript
// test-config.ts - Configuration update based on test mode
export async function instantUpdate(): Promise<void> {
  $("#testConfig .mode .textButton").removeClass("active");
  $("#testConfig .mode .textButton[mode='" + Config.mode + "']").addClass("active");

  if (Config.mode === "time") {
    $("#testConfig .time").removeClass("hidden");
    updateActiveExtraButtons("time", Config.time);
  } else if (Config.mode === "words") {
    $("#testConfig .wordCount").removeClass("hidden");
    updateActiveExtraButtons("words", Config.words);
  } else if (Config.mode === "quote") {
    $("#testConfig .quoteLength").removeClass("hidden");
  }
}
```

### Test Initialization

**File:** `frontend/src/ts/test/test-logic.ts` (Line 414)

**Function:** `init()`

```typescript
async function init(): Promise<boolean> {
  TestWords.words.reset();  // Clear previous words
  Loader.show();

  // Load language data
  const { data: language, error } = await tryCatch(
    JSONData.getLanguage(Config.language)
  );

  // Activate funboxes if applicable
  if (ActivePage.get() === "test") {
    await Funbox.activate();
  }

  // Generate words
  try {
    const gen = await WordsGenerator.generateWords(language);
    generatedWords = gen.words;
    generatedSectionIndexes = gen.sectionIndexes;
    wordsHaveTab = gen.hasTab;
    wordsHaveNewline = gen.hasNewline;
  } catch (e) {
    // Handle generation error
  }

  // UI updates
  Loader.hide();
  return true;
}
```

### Word Generation

**File:** `frontend/src/ts/test/words-generator.ts` (Line 608)

**Word Sources:**
- **Time/Words Mode**: Random selection from language word list
- **Quote Mode**: Fetched from QuotesController (random or filtered by length)
- **Custom Mode**: User-provided custom text
- **Zen Mode**: Generated dynamically as user types

**Generation Logic:**

```typescript
export async function generateWords(
  language: LanguageObject
): Promise<GenerateWordsReturn> {
  let wordList = language.words;

  if (Config.mode === "custom") {
    wordList = CustomText.getText();
  } else if (Config.mode === "quote") {
    wordList = await getQuoteWordList(language, wordOrder);
  } else if (Config.mode === "zen") {
    wordList = [];
  }

  // Apply word order (normal, reverse)
  if (wordOrder === "reverse") {
    wordList = wordList.reverse();
  }

  // Create wordset and generate words up to limit
  currentWordset = await withWords(wordList);

  while (!stop) {
    const nextWord = await getNextWord(i, limit, ...);
    ret.words.push(nextWord.word);
    if (ret.words.length >= limit) stop = true;
  }

  return ret;
}
```

**Punctuation Addition:**

```typescript
export async function punctuateWord(
  previousWord: string,
  currentWord: string,
  index: number
): Promise<string> {
  // Applies language-specific punctuation based on position
  // 10% chance for comma/period
  // Special handling for Spanish (¿?), French, etc.
}
```

---

## 2. During Test

### Test Start

**File:** `frontend/src/ts/test/test-logic.ts` (Line 103)

**Function:** `startTest()`

```typescript
export function startTest(now: number): boolean {
  if (PageTransition.get()) return false;

  TestState.setActive(true);
  Replay.startReplayRecording();  // Start recording for replay feature
  TestInput.resetKeypressTimings();

  // Show UI elements
  TimerProgress.show();
  LiveSpeed.show();
  LiveAcc.show();
  LiveBurst.show();
  Monkey.show();

  // Set timer start
  TestStats.setStart(now);
  void TestTimer.start();  // Start countdown/timer

  return true;
}
```

### User Input Capture

**File:** `frontend/src/ts/test/test-input.ts` (Line 97+)

The `Input` class manages user input:

```typescript
class Input {
  current: string;        // Current word being typed
  private history: string[];  // All typed words
  koreanStatus: boolean;  // For Korean input composition
}
```

**Key Events Tracked:**
- **Keypress timing**:
  - `spacing` - Time between keypresses (milliseconds)
  - `duration` - How long each key is held
- **Character accuracy**: correct, incorrect, missed, extra characters
- **Word submission**: Tracked when space is pressed

**Metrics Collected Per Second:**
- `keypressTimings.spacing.array` - Array of inter-key timings
- `keypressTimings.duration.array` - Array of key press durations
- `keyOverlap` - Detection of simultaneous key presses
- `afkHistory` - AFK detection array (one entry per second)
- `accuracy` - Running count: `{ correct, incorrect }`

### Real-time Metrics Calculation

**File:** `frontend/src/ts/test/test-stats.ts`

#### WPM Calculation (Line 147)

```typescript
export function calculateWpmAndRaw(): { wpm: number; raw: number } {
  const testSeconds = calculateTestSeconds(
    TestState.isActive ? performance.now() : end
  );
  const chars = countChars();

  // WPM = (correct chars + correct spaces) * (60 / seconds) / 5
  const wpm = ((chars.correctWordChars + chars.correctSpaces) *
               (60 / testSeconds)) / 5;

  // Raw = (all typed chars) * (60 / seconds) / 5
  const raw = ((chars.allCorrectChars + chars.spaces +
                chars.incorrectChars + chars.extraChars) *
               (60 / testSeconds)) / 5;

  return { wpm: Math.round(wpm), raw: Math.round(raw) };
}
```

#### Accuracy Calculation (Line 227)

```typescript
export function calculateAccuracy(): number {
  const acc = (TestInput.accuracy.correct /
    (TestInput.accuracy.correct + TestInput.accuracy.incorrect)) * 100;
  return isNaN(acc) ? 100 : acc;
}
```

#### Burst Speed Calculation (Line 209)

```typescript
export function calculateBurst(): number {
  const timeToWrite = (performance.now() - TestInput.currentBurstStart) / 1000;
  let wordLength = TestInput.input.current.length;
  const speed = (wordLength * (60 / timeToWrite)) / 5;
  return Math.round(speed);
}
```

#### AFK Detection (Line 184)

```typescript
export function calculateAfkSeconds(testSeconds: number): number {
  let extraAfk = 0;
  // Calculate missing keypresses per second
  extraAfk = Math.round(testSeconds) - TestInput.keypressCountHistory.length;
  if (extraAfk < 0) extraAfk = 0;

  const ret = TestInput.afkHistory.filter((afk) => afk).length;
  return ret + extraAfk;
}
```

### Data Tracked During Test

```typescript
TestInput tracks per second:
- wpmHistory[]              // WPM for each second
- rawHistory[]              // Raw WPM for each second
- burstHistory[]            // Burst speed for each second
- errorHistory[]            // Error count per second
- keypressCountHistory[]    // Keypresses per second
- afkHistory[]              // AFK status per second (boolean array)
- missedWords[]             // Words started but not completed
- accuracy                  // Running accuracy: { correct, incorrect }

TestStats tracks:
- start, end                // performance.now() timestamps
- start3, end3              // Date.getTime() timestamps
- lastSecondNotRound        // Whether test duration has decimal seconds
```

---

## 3. Test Completion

### Test End Trigger

**File:** `frontend/src/ts/test/test-logic.ts` (Line 915)

**Function:** `finish()`

**Trigger Points:**
1. Timer reaches 0 (time mode)
2. Word count reached (words mode)
3. Quote completed (quote mode)
4. Custom text limit reached (custom mode)
5. User manually stops test (any mode)

```typescript
export async function finish(difficultyFailed = false): Promise<void> {
  if (!TestState.isActive) return;

  TestUI.setResultCalculating(true);
  const now = performance.now();
  TestStats.setEnd(now);  // Record end time

  await Misc.sleep(1);  // Ensure last keypress is registered

  // Push last word to history if not empty
  if (TestInput.input.current.length !== 0) {
    TestInput.input.pushHistory();
    TestInput.corrected.pushHistory();
  }

  TestInput.forceKeyup(now);  // Register final keypresses

  // Hide UI elements
  TestState.setResultVisible(true);
  TestState.setActive(false);
  Caret.hide();
  LiveSpeed.hide();
  LiveAcc.hide();
  LiveBurst.hide();
  TimerProgress.hide();
  Monkey.hide();

  // Continue to stats calculation...
}
```

### Result Construction

**File:** `frontend/src/ts/test/test-logic.ts` (Line 777)

**Function:** `buildCompletedEvent()`

**CompletedEvent Structure:**

```typescript
const completedEvent: CompletedEvent = {
  // Core metrics
  wpm: stats.wpm,                    // Final WPM
  rawWpm: stats.wpmRaw,              // Final raw WPM
  charStats: [
    correctChars + correctSpaces,    // [0] Correct
    incorrectChars,                  // [1] Incorrect
    extraChars,                      // [2] Extra
    missedChars                      // [3] Missed
  ],
  charTotal: stats.allChars,
  acc: stats.acc,                    // Final accuracy %

  // Test configuration
  mode: Config.mode,                 // time, words, quote, custom, zen
  mode2: Misc.getMode2(Config, quote), // 60, 10, etc.
  punctuation: Config.punctuation,
  numbers: Config.numbers,
  lazyMode: Config.lazyMode,
  difficulty: Config.difficulty,
  blindMode: Config.blindMode,
  language: language,
  funbox: Config.funbox,

  // Metadata
  timestamp: Date.now(),
  tags: activeTagsIds,               // Active tag IDs

  // Anti-cheat data
  restartCount: TestStats.restartCount,
  incompleteTests: TestStats.incompleteTests, // Array of incomplete tests
  incompleteTestSeconds: TestStats.incompleteSeconds,

  // Keystroke data
  keySpacing: TestInput.keypressTimings.spacing.array,
  keyDuration: TestInput.keypressTimings.duration.array,
  keyOverlap: TestInput.keyOverlap.total,

  // Timing data
  lastKeyToEnd: lkte,                // Time from last key to test end
  startToFirstKey: stfk,             // Time from test start to first key
  testDuration: duration,
  afkDuration: afkDuration,

  // Consistency metrics
  consistency: consistency,           // Speed consistency (kogasa formula)
  wpmConsistency: wpmConsistency,    // WPM consistency
  keyConsistency: keyConsistency,    // Key timing consistency

  // Additional data
  chartData: {
    wpm: TestInput.wpmHistory,
    burst: rawPerSecond,
    err: errorHistory
  },
  customText: customText,            // For custom mode
  bailedOut: TestState.bailedOut,
  stopOnLetter: Config.stopOnError === "letter"
};
```

### Frontend Validation

**File:** `frontend/src/ts/test/test-logic.ts` (Line 1025+)

**Validation Checks:**

```typescript
// Test duration consistency (time mode only)
if (Config.mode === "time" && ce.testDuration < dateDur - 0.1) {
  // Mark invalid: inconsistent test duration
}

// Test too short
if (completedEvent.testDuration < 1) {
  dontSave = true;
}

// WPM sanity checks
if (completedEvent.wpm > 350) {
  dontSave = true;  // Too high
}

// Accuracy bounds
if (completedEvent.acc < 75 || completedEvent.acc > 100) {
  // Mark invalid (unless leaderboard opt-out)
}

// AFK detection
const kps = TestInput.afkHistory.slice(-5);
let afkDetected = kps.every((afk) => afk);  // All of last 5 seconds AFK

// Repeated test check
if (TestState.isRepeated) {
  dontSave = true;
}
```

### Result Display

**File:** `frontend/src/ts/test/result.ts`

The result page displays:
- Speed chart (WPM, Raw, Burst over time)
- Accuracy metrics
- Key statistics (consistency, errors, etc.)
- Consistency data
- If personal best: Crown animation and confetti

---

## 4. Result Submission

### Frontend API Call

**File:** `frontend/src/ts/test/test-logic.ts` (Line 1276)

**Function:** `saveResult()`

```typescript
async function saveResult(
  completedEvent: CompletedEvent,
  isRetrying: boolean
): Promise<void> {
  // Check if result saving is enabled
  if (!TestState.savingEnabled) {
    Notifications.add("Result not saved: disabled by user", -1);
    return;
  }

  // Check connection
  if (!ConnectionState.get()) {
    Notifications.add("Result not saved: offline", -1);
    retrySaving.canRetry = true;
    retrySaving.completedEvent = completedEvent;
    return;
  }

  // Calculate hash for integrity verification
  completedEvent.hash = objectHash(completedEvent);

  // Send to API
  const response = await Ape.results.add({
    body: { result: completedEvent }
  });

  // Handle response
  if (response.status !== 200) {
    // Allow retry unless specific error codes: 460, 461, 463-466
    if (![460, 461, 463, 464, 465, 466].includes(response.status)) {
      retrySaving.canRetry = true;
    }
  }
}
```

### API Endpoint

**Contract:** `packages/contracts/src/results.ts`

```
POST /results
Content-Type: application/json

Request Body:
{
  "result": CompletedEvent
}

Response Codes:
- 200: Success (PostResultResponse)
- 460: Test too short
- 461: Result hash invalid
- 462: Result spacing invalid
- 463: Result data invalid
- 464: Missing key data
- 465: Bot detected
- 466: Duplicate result
```

### Success Response

**Response Data (PostResultResponse):**

```typescript
{
  insertedId: string,                // MongoDB ObjectId as hex string
  xp: number,                        // XP gained
  dailyXpBonus?: boolean,            // Daily bonus awarded?
  xpBreakdown?: XpBreakdown,         // XP source breakdown
  streak?: number,                   // Current streak
  isPb?: boolean,                    // Is personal best?
  tagPbs?: string[],                 // Tag personal bests
  dailyLeaderboardRank?: number,     // Daily leaderboard rank
  weeklyXpLeaderboardRank?: number   // Weekly XP leaderboard rank
}
```

### Client Response Handling

**File:** `frontend/src/ts/test/test-logic.ts` (Line 1336+)

```typescript
const data = response.body.data;

// Save result ID for tagging
$("#result .stats .tags .editTagsButton").attr(
  "data-result-id",
  data.insertedId
);

// Update XP display
if (data.xp !== undefined) {
  void XPBar.update(snapxp, data.xp, data.xpBreakdown);
}

// Update streak
if (data.streak !== undefined) {
  dataToSave.streak = data.streak;
}

// Check for PB
if (data.isPb !== undefined && data.isPb) {
  Result.showConfetti();
  Result.showCrown("normal");
}

// Store in local DB
void DB.saveLocalResultData(dataToSave);
```

---

## 5. Backend Processing

### Route Handler

**File:** `backend/src/api/routes/results.ts`

```typescript
export default s.router(resultsContract, {
  add: {
    handler: async (r) => callController(ResultController.addResult)(r),
  },
});
```

**Endpoint:** `POST /results`

### Main Controller Logic

**File:** `backend/src/api/controllers/result.ts` (Line 199)

**Function:** `addResult()`

The backend performs a comprehensive validation and processing pipeline:

#### ① User & Basic Validation (Line 204-223)

```typescript
const { uid } = req.ctx.decodedToken;
const user = await UserDAL.getUser(uid, "add result");

if (user.needsToChangeName) {
  throw new MonkeyError(403, "Please change your name first");
}

const completedEvent = req.body.result;
completedEvent.uid = uid;

// Check if test is too short
if (isTestTooShort(completedEvent)) {
  throw new MonkeyError(460, "Test too short");
}

// Accuracy check (if not opted out of leaderboards)
if (user.lbOptOut !== true && completedEvent.acc < 75) {
  throw new MonkeyError(400, "Accuracy too low");
}
```

#### ② Hash Validation (Line 225-244)

```typescript
const resulthash = completedEvent.hash;

if (req.ctx.configuration.results.objectHashCheckEnabled) {
  const objectToHash = omit(completedEvent, "hash");
  const serverhash = objectHash(objectToHash);

  if (serverhash !== resulthash) {
    void addLog("incorrect_result_hash",
      { serverhash, resulthash, result: completedEvent }, uid);
    throw new MonkeyError(461, "Incorrect result hash");
  }
}
```

#### ③ Funbox Validation (Line 246-252)

```typescript
if (completedEvent.funbox.length !== _.uniq(completedEvent.funbox).length) {
  throw new MonkeyError(400, "Duplicate funboxes");
}

if (!checkCompatibility(completedEvent.funbox)) {
  throw new MonkeyError(400, "Impossible funbox combination");
}
```

#### ④ Key Statistics Calculation (Line 254-280)

```typescript
let keySpacingStats: KeyStats | undefined = undefined;
if (completedEvent.keySpacing !== "toolong" &&
    completedEvent.keySpacing.length > 0) {
  keySpacingStats = {
    average: completedEvent.keySpacing.reduce((a, b) => a + b) /
             completedEvent.keySpacing.length,
    sd: stdDev(completedEvent.keySpacing)
  };
}

let keyDurationStats: KeyStats | undefined = undefined;
if (completedEvent.keyDuration !== "toolong" &&
    completedEvent.keyDuration.length > 0) {
  keyDurationStats = {
    average: completedEvent.keyDuration.reduce((a, b) => a + b) /
             completedEvent.keyDuration.length,
    sd: stdDev(completedEvent.keyDuration)
  };
}
```

#### ⑤ Anti-Cheat Validation (Line 295-317)

```typescript
if (anticheatImplemented()) {
  // Validate result data makes sense
  if (!validateResult(
    completedEvent,
    req.raw.headers["x-client-version"],
    JSON.stringify(new UAParser(req.raw.headers["user-agent"]).getResult()),
    user.lbOptOut === true
  )) {
    const status = MonkeyStatusCodes.RESULT_DATA_INVALID;
    throw new MonkeyError(status.code, "Result data doesn't make sense");
  }
}

// Keystroke pattern validation for high WPM
if (completedEvent.mode === "time" &&
    completedEvent.wpm > 130 &&
    completedEvent.testDuration < 122 &&
    (user.verified === false || user.verified === undefined) &&
    user.lbOptOut !== true) {

  if (!validateKeys(completedEvent, keySpacingStats, keyDurationStats, uid)) {
    // Auto-ban if enabled
    if (autoBanConfig.enabled) {
      const didUserGetBanned = await UserDAL.recordAutoBanEvent(
        uid, autoBanConfig.maxCount, autoBanConfig.maxHours);
      if (didUserGetBanned) {
        user.banned = true;
      }
    }
    throw new MonkeyError(465, "Possible bot detected");
  }
}
```

#### ⑥ Result Spacing Validation (Line 331-361)

```typescript
const lastResultTimestamp = await ResultDAL.getLastResultTimestamp(uid);

completedEvent.timestamp = Math.floor(Date.now() / 1000) * 1000;

// Check if enough time has passed since last result
const testDurationMilis = completedEvent.testDuration * 1000;
const incompleteTestsMilis = completedEvent.incompleteTestSeconds * 1000;
const earliestPossible =
  (lastResultTimestamp ?? 0) + testDurationMilis + incompleteTestsMilis;
const nowNoMilis = Math.floor(Date.now() / 1000) * 1000;

if (isSafeNumber(lastResultTimestamp) &&
    nowNoMilis < earliestPossible - 1000) {
  void addLog("invalid_result_spacing",
    { lastResultTimestamp, testDuration, incompleteTestSeconds, now: nowNoMilis },
    uid);
  throw new MonkeyError(462, "Invalid result spacing");
}
```

#### ⑦ Duplicate Detection (Line 416-438)

```typescript
if (req.ctx.configuration.users.lastHashesCheck.enabled) {
  let lastHashes = user.lastReultHashes ?? [];

  if (lastHashes.includes(resulthash)) {
    void addLog("duplicate_result", { resulthash }, uid);
    throw new MonkeyError(466, "Duplicate result");
  }

  lastHashes.unshift(resulthash);
  const maxHashes = req.ctx.configuration.users.lastHashesCheck.maxHashes;
  if (lastHashes.length > maxHashes) {
    lastHashes = lastHashes.slice(0, maxHashes);
  }
  await UserDAL.updateLastHashes(uid, lastHashes);
}
```

#### ⑧ Personal Best Checking (Line 449-457)

```typescript
let isPb = false;
let tagPbs: string[] = [];

if (!completedEvent.bailedOut) {
  [isPb, tagPbs] = await Promise.all([
    UserDAL.checkIfPb(uid, user, completedEvent),
    UserDAL.checkIfTagPb(uid, user, completedEvent),
  ]);
}
```

#### ⑨ Discord Integration (Line 459-481)

```typescript
// For 60 second time mode
if (completedEvent.mode === "time" && completedEvent.mode2 === "60") {
  void UserDAL.incrementBananas(uid, completedEvent.wpm);

  if (isPb && user.discordId && user.lbOptOut !== true) {
    void GeorgeQueue.updateDiscordRole(user.discordId, completedEvent.wpm);
  }
}

// Challenge completion
if (completedEvent.challenge &&
    AutoRoleList.includes(completedEvent.challenge) &&
    user.discordId) {
  void GeorgeQueue.awardChallenge(user.discordId, completedEvent.challenge);
}
```

#### ⑩ Stats Updates (Line 486-494)

```typescript
const afk = completedEvent.afkDuration ?? 0;
const totalDurationTypedSeconds =
  completedEvent.testDuration + completedEvent.incompleteTestSeconds - afk;

// Update user typing stats
void UserDAL.updateTypingStats(
  uid,
  completedEvent.restartCount,
  totalDurationTypedSeconds
);

// Update global public stats
void PublicDAL.updateStats(
  completedEvent.restartCount,
  totalDurationTypedSeconds
);
```

#### ⑪ Daily Leaderboard Updates (Line 496-561)

```typescript
const dailyLeaderboard = getDailyLeaderboard(
  completedEvent.language,
  completedEvent.mode,
  completedEvent.mode2,
  dailyLeaderboardsConfig
);

let dailyLeaderboardRank = -1;

if (dailyLeaderboard && validResultCriteria) {
  dailyLeaderboardRank = await dailyLeaderboard.addResult(
    {
      name: user.name,
      wpm: completedEvent.wpm,
      raw: completedEvent.rawWpm,
      acc: completedEvent.acc,
      consistency: completedEvent.consistency,
      timestamp: completedEvent.timestamp,
      uid,
      discordAvatar: user.discordAvatar,
      discordId: user.discordId,
      badgeId: selectedBadgeId,
      isPremium,
    },
    dailyLeaderboardsConfig
  );
}
```

#### ⑫ Streak & Badge Updates (Line 563-589)

```typescript
const streak = await UserDAL.updateStreak(uid, completedEvent.timestamp);

// Check if user should get 365-day streak badge
const shouldGetBadge =
  streak >= 365 &&
  !user.inventory?.badges?.find(b => b.id === 14) &&
  !badgeWaitingInInbox;

if (shouldGetBadge) {
  const mail = buildMonkeyMail({
    subject: "Badge",
    body: "Congratulations for reaching a 365 day streak!",
    rewards: [{ type: "badge", item: { id: 14 } }],
  });
  await UserDAL.addToInbox(uid, [mail], req.ctx.configuration.users.inbox);
}
```

#### ⑬ XP Calculation (Line 591-610)

```typescript
const xpGained = await calculateXp(
  completedEvent,
  req.ctx.configuration.users.xp,
  lastResultTimestamp,
  user.xp ?? 0,
  streak
);

if (xpGained.xp < 0) {
  throw new MonkeyError(500, "Calculated XP is negative");
}
```

**XP Calculation Logic (Line 690-841):**

```typescript
async function calculateXp(
  result: CompletedEvent,
  xpConfiguration: Configuration["users"]["xp"],
  lastResultTimestamp: number | null,
  currentTotalXp: number,
  streak: number
): Promise<XpResult> {
  if (result.mode === "zen" || !enabled) {
    return { xp: 0 };
  }

  const breakdown: XpBreakdown = {};

  // Base XP: 2 XP per second of actual typing
  const baseXp = Math.round((result.testDuration - result.afkDuration) * 2);
  breakdown.base = baseXp;

  let modifier = 1;

  // Bonuses
  if (result.acc === 100) {
    modifier += 0.5;  // 50% bonus for perfect accuracy
    breakdown.fullAccuracy = Math.round(baseXp * 0.5);
  }

  if (result.mode === "quote") {
    modifier += 0.5;  // 50% bonus for quotes
    breakdown.quote = Math.round(baseXp * 0.5);
  } else {
    if (result.punctuation) {
      modifier += 0.4;  // 40% bonus for punctuation
      breakdown.punctuation = Math.round(baseXp * 0.4);
    }
    if (result.numbers) {
      modifier += 0.1;  // 10% bonus for numbers
      breakdown.numbers = Math.round(baseXp * 0.1);
    }
  }

  // Funbox bonus
  if (funboxBonusConfiguration > 0 && result.funbox.length !== 0) {
    const funboxModifier = _.sumBy(result.funbox, (funboxName) => {
      const funbox = getFunbox(funboxName);
      return Math.max((funbox?.difficultyLevel ?? 0) *
                      funboxBonusConfiguration, 0);
    });
    if (funboxModifier > 0) {
      modifier += funboxModifier;
      breakdown.funbox = Math.round(baseXp * funboxModifier);
    }
  }

  // Streak multiplier
  if (xpConfiguration.streak.enabled) {
    const streakModifier = mapRange(
      streak,
      0,
      xpConfiguration.streak.maxStreakDays,
      0,
      xpConfiguration.streak.maxStreakMultiplier,
      true
    );
    if (streakModifier > 0) {
      modifier += streakModifier;
      breakdown.streak = Math.round(baseXp * streakModifier);
    }
  }

  // Incomplete tests bonus
  let incompleteXp = 0;
  if (result.incompleteTests?.length > 0) {
    result.incompleteTests.forEach((it) => {
      let modifier = (it.acc - 50) / 50;
      if (modifier < 0) modifier = 0;
      incompleteXp += Math.round(it.seconds * modifier);
    });
    breakdown.incomplete = incompleteXp;
  }

  // Accuracy penalty/bonus
  const accuracyModifier = (result.acc - 50) / 50;
  breakdown.accPenalty = Math.round(baseXp * modifier * (accuracyModifier - 1));

  // Daily bonus (5% of total XP, with min/max bounds)
  let dailyBonus = 0;
  if (isSafeNumber(lastResultTimestamp)) {
    const lastResultDay = getStartOfDayTimestamp(lastResultTimestamp);
    const today = getCurrentDayTimestamp();
    if (lastResultDay !== today) {
      const proportionalXp = Math.round(currentTotalXp * 0.05);
      dailyBonus = Math.max(
        Math.min(xpConfiguration.maxDailyBonus, proportionalXp),
        xpConfiguration.minDailyBonus
      );
      breakdown.daily = dailyBonus;
    }
  }

  // Final calculation
  const xpWithModifiers = Math.round(baseXp * modifier);
  const xpAfterAccuracy = Math.round(xpWithModifiers * accuracyModifier);
  const totalXp = Math.round((xpAfterAccuracy + incompleteXp) *
                             gainMultiplier) + dailyBonus;

  return { xp: totalXp, dailyBonus: dailyBonus > 0, breakdown };
}
```

**XP Modifiers Summary:**
- **Base**: 2 XP per second typed (minus AFK time)
- **100% Accuracy**: +50%
- **Quote mode**: +50%
- **Punctuation**: +40%
- **Numbers**: +10%
- **Funbox**: Variable based on difficulty
- **Streak**: Up to configured max multiplier
- **Accuracy**: Linear scale from 50% (0x) to 100% (1x)
- **Daily bonus**: 5% of total XP (once per day, min/max capped)

#### ⑭ Weekly XP Leaderboard (Line 611-634)

```typescript
const weeklyXpLeaderboard = WeeklyXpLeaderboard.get(weeklyXpLeaderboardConfig);

if (userEligibleForLeaderboard && xpGained.xp > 0 && weeklyXpLeaderboard) {
  weeklyXpLeaderboardRank = await weeklyXpLeaderboard.addResult(
    weeklyXpLeaderboardConfig,
    {
      entry: {
        uid,
        name: user.name,
        discordAvatar: user.discordAvatar,
        discordId: user.discordId,
        badgeId: selectedBadgeId,
        lastActivityTimestamp: Date.now(),
        isPremium,
        timeTypedSeconds: totalDurationTypedSeconds,
      },
      xpGained: xpGained.xp,
    }
  );
}
```

#### ⑮ Save to MongoDB (Line 636-647)

```typescript
const dbresult = buildDbResult(completedEvent, user.name, isPb);

if (keySpacingStats !== undefined) {
  dbresult.keySpacingStats = keySpacingStats;
}
if (keyDurationStats !== undefined) {
  dbresult.keyDurationStats = keyDurationStats;
}

const addedResult = await ResultDAL.addResult(uid, dbresult);

await UserDAL.incrementXp(uid, xpGained.xp);
await UserDAL.incrementTestActivity(user, completedEvent.timestamp);
```

### Data Access Layer

**File:** `backend/src/dal/result.ts`

```typescript
export async function addResult(
  uid: string,
  result: DBResult
): Promise<{ insertedId: ObjectId }> {
  const { data: user } = await tryCatch(getUser(uid, "add result"));

  if (!user) throw new MonkeyError(404, "User not found");
  if (result.uid === undefined) result.uid = uid;

  // Insert into MongoDB
  const res = await getResultCollection().insertOne(result);

  return { insertedId: res.insertedId };
}
```

**MongoDB Collection:** `results`

**Result Document Structure:**

```javascript
{
  _id: ObjectId,
  uid: string,
  wpm: number,
  rawWpm: number,
  charStats: [correct, incorrect, extra, missed],
  charTotal: number,
  acc: number,
  mode: string,
  mode2: string,
  punctuation: boolean,
  numbers: boolean,
  lazyMode: boolean,
  timestamp: number,
  language: string,
  restartCount: number,
  incompleteTests: [{acc: number, seconds: number}],
  incompleteTestSeconds: number,
  difficulty: string,
  blindMode: boolean,
  tags: [tagId1, tagId2],
  keySpacingStats: {average: number, sd: number},
  keyDurationStats: {average: number, sd: number},
  keySpacing: [milliseconds array],
  keyDuration: [milliseconds array],
  keyOverlap: number,
  lastKeyToEnd: number,
  startToFirstKey: number,
  consistency: number,
  wpmConsistency: number,
  keyConsistency: number,
  funbox: [funboxName1, funboxName2],
  bailedOut: boolean,
  chartData: {wpm: [], burst: [], err: []},
  customText: {textLen, mode, limit, pipeDelimiter},
  testDuration: number,
  afkDuration: number,
  isPb: boolean,
  name: string  // Username snapshot
}
```

### Response Construction

**File:** `backend/src/api/controllers/result.ts` (Line 661-681)

```typescript
const data: PostResultResponse = {
  isPb,
  tagPbs,
  insertedId: addedResult.insertedId.toHexString(),
  xp: xpGained.xp,
  dailyXpBonus: xpGained.dailyBonus ?? false,
  xpBreakdown: xpGained.breakdown ?? {},
  streak,
};

if (dailyLeaderboardRank !== -1) {
  data.dailyLeaderboardRank = dailyLeaderboardRank;
}

if (weeklyXpLeaderboardRank !== -1) {
  data.weeklyXpLeaderboardRank = weeklyXpLeaderboardRank;
}

incrementResult(completedEvent, dbresult.isPb);  // Prometheus metrics

return new MonkeyResponse("Result saved", data);
```

---

## Flow Diagram

```
┌─────────────────────────────────────┐
│      START TEST SESSION             │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Configuration Selection            │
│  - Mode (time/words/quote/custom)   │
│  - Options (punctuation/numbers)    │
│  - Language, difficulty, funboxes   │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Word Generation                    │
│  - Time/Words: Random from language │
│  - Quote: Fetch from database       │
│  - Custom: User text                │
│  - Apply punctuation & numbers      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Display Test UI                    │
│  - Render words                     │
│  - Show timer/progress bar          │
│  - Initialize input capture         │
└──────────────┬──────────────────────┘
               │
               ▼
╔═════════════════════════════════════╗
║     USER TYPING (Real-time)         ║
╠═════════════════════════════════════╣
║ Per Keypress:                       ║
║  - Capture timing (spacing/duration)║
║  - Validate character accuracy      ║
║  - Detect simultaneous keypresses   ║
║                                     ║
║ Per Second:                         ║
║  - Calculate WPM & Raw WPM          ║
║  - Calculate burst speed            ║
║  - Track errors                     ║
║  - Detect AFK                       ║
║  - Record to history arrays         ║
╚═════════════════╤═══════════════════╝
                  │
                  ▼
┌─────────────────────────────────────┐
│  Test Completion Trigger            │
│  - Timer expires OR                 │
│  - Word count reached OR            │
│  - Manual stop                      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Calculate Final Statistics         │
│  - Final WPM, Raw WPM, Accuracy     │
│  - Consistency metrics              │
│  - Character statistics             │
│  - AFK duration                     │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Build CompletedEvent Object        │
│  - Core metrics                     │
│  - Test configuration               │
│  - Keystroke data arrays            │
│  - Chart data                       │
│  - Anti-cheat metadata              │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Frontend Validation                │
│  - Test duration ≥ 1s               │
│  - WPM ≤ 350                        │
│  - Accuracy 75-100%                 │
│  - Not repeated test                │
│  - AFK check                        │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Calculate Result Hash              │
│  hash = objectHash(completedEvent)  │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  API Request                        │
│  POST /results                      │
│  { "result": completedEvent }       │
└──────────────┬──────────────────────┘
               │
               ▼
╔═════════════════════════════════════╗
║   BACKEND VALIDATION PIPELINE       ║
╠═════════════════════════════════════╣
║ ① User & Basic Validation           ║
║    - User exists                    ║
║    - Test duration check            ║
║    - Accuracy ≥ 75%                 ║
║                                     ║
║ ② Hash Validation                   ║
║    - Recalculate hash server-side   ║
║    - Compare with client hash       ║
║                                     ║
║ ③ Funbox Validation                 ║
║    - No duplicates                  ║
║    - Valid combinations             ║
║                                     ║
║ ④ Calculate Key Statistics          ║
║    - Average spacing & duration     ║
║    - Standard deviation             ║
║                                     ║
║ ⑤ Anti-Cheat Checks                 ║
║    - Result data validation         ║
║    - Keystroke pattern analysis     ║
║    - Bot detection (high WPM)       ║
║    - Auto-ban if needed             ║
║                                     ║
║ ⑥ Result Spacing Validation         ║
║    - Check time since last result   ║
║    - Prevent impossible timing      ║
║                                     ║
║ ⑦ Duplicate Detection               ║
║    - Check against last N hashes    ║
║    - Prevent resubmission           ║
╚═════════════════╤═══════════════════╝
                  │
                  ▼
╔═════════════════════════════════════╗
║   BACKEND PROCESSING PIPELINE       ║
╠═════════════════════════════════════╣
║ ⑧ Personal Best Check                ║
║    - Overall PB                     ║
║    - Tag-specific PBs               ║
║                                     ║
║ ⑨ Discord Integration               ║
║    - Update Discord role (60s mode) ║
║    - Award challenge badges         ║
║                                     ║
║ ⑩ Stats Updates                     ║
║    - User typing stats              ║
║    - Global public stats            ║
║                                     ║
║ ⑪ Daily Leaderboard                 ║
║    - Add to language/mode board     ║
║    - Calculate rank                 ║
║                                     ║
║ ⑫ Streak & Badge Updates            ║
║    - Update streak                  ║
║    - Award 365-day badge            ║
║                                     ║
║ ⑬ XP Calculation                    ║
║    - Base: 2 XP/second              ║
║    - Modifiers: accuracy, mode, etc.║
║    - Daily bonus (5% of total XP)   ║
║                                     ║
║ ⑭ Weekly XP Leaderboard             ║
║    - Add XP to weekly total         ║
║    - Calculate rank                 ║
╚═════════════════╤═══════════════════╝
                  │
                  ▼
┌─────────────────────────────────────┐
│  Build DB Result Document           │
│  - Core metrics                     │
│  - Key statistics                   │
│  - Chart data                       │
│  - Metadata                         │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Save to MongoDB                    │
│  results.insertOne(dbresult)        │
│  ↓                                  │
│  { insertedId: ObjectId }           │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Update User Document               │
│  - Increment XP                     │
│  - Update test activity             │
│  - Update last hash list            │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Construct API Response             │
│  {                                  │
│    insertedId,                      │
│    xp, xpBreakdown, dailyXpBonus,   │
│    streak,                          │
│    isPb, tagPbs,                    │
│    dailyLeaderboardRank,            │
│    weeklyXpLeaderboardRank          │
│  }                                  │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Return 200 OK to Frontend          │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Frontend Receives Response         │
│  - Save result ID                   │
│  - Update XP display (animated)     │
│  - Update streak display            │
│  - Show confetti if PB              │
│  - Save to local IndexedDB          │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Display Result Screen              │
│  - Speed chart                      │
│  - Accuracy & consistency           │
│  - Detailed statistics              │
│  - Crown if PB                      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Ready for Next Test                │
└─────────────────────────────────────┘
```

---

## Anti-Cheat Measures

Monkeytype implements multiple layers of anti-cheat protection:

### 1. Hash Validation

**Location:** `backend/src/api/controllers/result.ts:225`

- Client calculates hash of result object using `objectHash()`
- Server recalculates the same hash independently
- Any mismatch indicates client-side tampering
- Logged for investigation

### 2. Result Spacing Validation

**Location:** `backend/src/api/controllers/result.ts:331`

- Tracks timestamp of last result submission
- Calculates minimum time required: `lastResult + testDuration + incompleteTime`
- Prevents submitting results faster than physically possible
- Accounts for incomplete tests and restarts

### 3. Keystroke Pattern Analysis

**Location:** `backend/src/api/controllers/result.ts:295`

**Triggered for:**
- WPM > 130
- Test duration < 122 seconds
- Unverified users
- Not opted out of leaderboards

**Validates:**
- Key spacing patterns (time between keypresses)
- Key duration patterns (how long keys are held)
- Statistical analysis of timing consistency
- Detects inhuman patterns

### 4. Duplicate Detection

**Location:** `backend/src/api/controllers/result.ts:416`

- Stores last N result hashes per user
- Checks if current hash matches any recent hash
- Prevents resubmitting the same result
- Configurable hash history size

### 5. Result Data Validation

**Location:** `backend/src/api/controllers/result.ts:295`

- Validates result metrics make sense mathematically
- Checks WPM calculation consistency
- Verifies accuracy calculations
- Validates test duration against mode
- Cross-references client version and user agent

### 6. Auto-Ban System

**Location:** `backend/src/api/controllers/result.ts:308`

- Triggered by multiple suspicious results
- Configurable threshold (e.g., 3 violations in 24 hours)
- Automatically bans user account
- Logged for admin review

### 7. Basic Sanity Checks

**Frontend:** `frontend/src/ts/test/test-logic.ts:1025`
**Backend:** `backend/src/api/controllers/result.ts:204`

- WPM ≤ 350 (frontend) / configurable (backend)
- Accuracy: 75-100%
- Test duration ≥ 1 second
- Valid test mode combinations
- No repeated tests (using same words)

### 8. Server-Side Timestamp

**Location:** `backend/src/api/controllers/result.ts:333`

- Server sets the timestamp, not client
- Prevents timestamp manipulation
- Used for streak and daily bonus calculations

---

## File References

### Frontend Files

| Component | File Path | Key Lines |
|-----------|-----------|-----------|
| Test Configuration | `frontend/src/ts/test/test-config.ts` | - |
| Test Initialization | `frontend/src/ts/test/test-logic.ts` | 414 |
| Word Generation | `frontend/src/ts/test/words-generator.ts` | 608 |
| Test Start | `frontend/src/ts/test/test-logic.ts` | 103 |
| Input Tracking | `frontend/src/ts/test/test-input.ts` | 97+ |
| WPM Calculation | `frontend/src/ts/test/test-stats.ts` | 147 |
| Accuracy Calculation | `frontend/src/ts/test/test-stats.ts` | 227 |
| Burst Calculation | `frontend/src/ts/test/test-stats.ts` | 209 |
| AFK Detection | `frontend/src/ts/test/test-stats.ts` | 184 |
| Test Finish | `frontend/src/ts/test/test-logic.ts` | 915 |
| Build Result | `frontend/src/ts/test/test-logic.ts` | 777 |
| Frontend Validation | `frontend/src/ts/test/test-logic.ts` | 1025 |
| Result Submission | `frontend/src/ts/test/test-logic.ts` | 1276 |
| Response Handling | `frontend/src/ts/test/test-logic.ts` | 1336 |
| Result Display | `frontend/src/ts/test/result.ts` | - |

### Backend Files

| Component | File Path | Key Lines |
|-----------|-----------|-----------|
| API Routes | `backend/src/api/routes/results.ts` | - |
| Main Controller | `backend/src/api/controllers/result.ts` | 199 |
| User Validation | `backend/src/api/controllers/result.ts` | 204 |
| Hash Validation | `backend/src/api/controllers/result.ts` | 225 |
| Anti-Cheat | `backend/src/api/controllers/result.ts` | 295 |
| Result Spacing | `backend/src/api/controllers/result.ts` | 331 |
| Duplicate Detection | `backend/src/api/controllers/result.ts` | 416 |
| PB Check | `backend/src/api/controllers/result.ts` | 449 |
| Stats Updates | `backend/src/api/controllers/result.ts` | 486 |
| Leaderboard Updates | `backend/src/api/controllers/result.ts` | 496 |
| Streak Updates | `backend/src/api/controllers/result.ts` | 563 |
| XP Calculation | `backend/src/api/controllers/result.ts` | 591, 690 |
| Weekly XP Leaderboard | `backend/src/api/controllers/result.ts` | 611 |
| Save to DB | `backend/src/api/controllers/result.ts` | 636 |
| Response Construction | `backend/src/api/controllers/result.ts` | 661 |
| Result DAL | `backend/src/dal/result.ts` | - |
| User DAL | `backend/src/dal/user.ts` | - |

### Shared Files

| Component | File Path |
|-----------|-----------|
| API Contracts | `packages/contracts/src/results.ts` |
| Result Schemas | `packages/schemas/src/results.ts` |
| User Schemas | `packages/schemas/src/users.ts` |

---

## Summary

The typing test lifecycle in Monkeytype is a sophisticated system with:

1. **Frontend**: Real-time input tracking, comprehensive metrics calculation, and client-side validation
2. **Backend**: Multi-layered validation pipeline, anti-cheat systems, and complex XP/reward calculations
3. **Security**: Hash verification, timing validation, keystroke analysis, and duplicate detection
4. **Features**: Personal bests, leaderboards, streaks, badges, XP system, and Discord integration
5. **Data**: Complete test data stored in MongoDB with detailed statistics and anti-cheat metadata

The entire flow takes approximately 1-3 seconds from test completion to result persistence, with most of the time spent on comprehensive validation and processing to ensure fair play and accurate leaderboards.
