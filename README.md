# Antigravity Gemini 3.5 Flash OpenCode Setup & Patch Guide

This guide outlines the step-by-step process of configuring OpenCode and patching the `opencode-antigravity-auth` plugin to successfully use the **Gemini 3.5 Flash** model with its thinking variants (Low, Medium, High).

---

## The Issue
By default, the Google Antigravity backend only exposes the `gemini-3.5-flash-low` tier variant. Requesting `gemini-3.5-flash` directly without a tier suffix, or selecting higher variants (like `Medium` or `High`), results in a **404: Requested entity was not found** error because the model names are not recognized by the backend.

To fix this, we:
1. Configured OpenCode to expose all variants in the model picker.
2. Patched the plugin's model registry to recognize `antigravity-gemini-3.5-flash`.
3. Patched the plugin's resolver logic to collapse all Gemini 3.5 Flash variants down to `-low` under the hood.

---

## Step-by-Step Configuration & Patching

### Step 1: Update Global OpenCode Configuration
**File path**: [opencode.json](file:///Users/shashanksahu/.config/opencode/opencode.json)

1. Make sure you are using the `@beta` version of the plugin in the `plugin` array:
   ```json
   "plugin": [
     "opencode-antigravity-auth@beta",
     ...
   ]
   ```
2. Add the `antigravity-gemini-3.5-flash` model and all variants under `provider.google.models`:
   ```json
   "antigravity-gemini-3.5-flash": {
     "name": "Gemini 3.5 Flash (Antigravity)",
     "limit": {
       "context": 1048576,
       "output": 65536
     },
     "modalities": {
       "input": [
         "text",
         "image",
         "pdf"
       ],
       "output": [
         "text"
       ]
     },
     "variants": {
       "minimal": {
         "thinkingLevel": "minimal"
       },
       "low": {
         "thinkingLevel": "low"
       },
       "medium": {
         "thinkingLevel": "medium"
       },
       "high": {
         "thinkingLevel": "high"
       }
     }
   }
   ```

---

### Step 2: Register the Model in Plugin Registry
**File path**: [models.js](file:///Users/shashanksahu/.cache/opencode/packages/opencode-antigravity-auth@beta/node_modules/opencode-antigravity-auth/dist/src/plugin/config/models.js)

Add the `antigravity-gemini-3.5-flash` model definitions with variants to the plugin's list:
```javascript
    "antigravity-gemini-3.5-flash": {
        name: "Gemini 3.5 Flash (Antigravity)",
        limit: { context: 1048576, output: 65536 },
        modalities: DEFAULT_MODALITIES,
        variants: {
            minimal: { thinkingLevel: "minimal" },
            low: { thinkingLevel: "low" },
            medium: { thinkingLevel: "medium" },
            high: { thinkingLevel: "high" },
        },
    },
```

---

### Step 3: Implement Tier Mapping in Model Resolver
**File path**: [model-resolver.js](file:///Users/shashanksahu/.cache/opencode/packages/opencode-antigravity-auth@beta/node_modules/opencode-antigravity-auth/dist/src/plugin/transform/model-resolver.js)

1. Map other tier suffixes down to `-low` in `MODEL_ALIASES`:
   ```javascript
   export const MODEL_ALIASES = {
       ...
       // Gemini 3.5 Flash - Antigravity backend currently exposes only the
       // `gemini-3.5-flash-low` variant (verified via fetchAvailableModels).
       // Map all other tiers down to -low so the request does not 500.
       "gemini-3.5-flash-minimal": "gemini-3.5-flash-low",
       "gemini-3.5-flash-medium": "gemini-3.5-flash-low",
       "gemini-3.5-flash-high": "gemini-3.5-flash-low",
       ...
   };
   ```
2. Define a regex helper to identify the Gemini 3.5 Flash family:
   ```javascript
   const GEMINI_FLASH_REQUIRES_TIER_REGEX = /^gemini-3\.5-flash/i;
   ```
3. Update `resolveModelWithTier` to resolve and force `-low` suffix:
   ```javascript
   export function resolveModelWithTier(requestedModel, options = {}) {
       ...
       const isGemini3Pro = isGemini3ProModel(modelWithoutQuota);
       const isGemini3Flash = isGemini3FlashModel(modelWithoutQuota);
       const flashRequiresTier = GEMINI_FLASH_REQUIRES_TIER_REGEX.test(modelWithoutQuota);
       
       let antigravityModel = modelWithoutQuota;
       if (skipAlias) {
           if (isGemini3Pro && !tier && !isImageModel) {
               antigravityModel = `${modelWithoutQuota}-low`;
           }
           else if (flashRequiresTier && !tier) {
               // 3.5-flash without tier -> default to -low (only supported tier today)
               antigravityModel = `${modelWithoutQuota}-low`;
           }
           else if (flashRequiresTier && tier) {
               // 3.5-flash currently only supports -low on the backend; force it
               antigravityModel = `${baseName}-low`;
           }
           else if (isGemini3Flash && tier) {
               antigravityModel = baseName;
           }
       }
       ...
   }
   ```

---

## Verification & Usage

Now you can run OpenCode using the Gemini 3.5 Flash model directly or by choosing any of the variants:

### Command Line Usage
```bash
# Default / Low variant
opencode run "your message" --model=google/antigravity-gemini-3.5-flash

# High variant (will automatically map to gemini-3.5-flash-low)
opencode run "your message" --model=google/antigravity-gemini-3.5-flash --variant=high
```

### IDE Integration
You can now select **Gemini 3.5 Flash (Low)**, **Gemini 3.5 Flash (Medium)**, or **Gemini 3.5 Flash (High)** in your IDE's model picker, and all of them will resolve and work properly.
