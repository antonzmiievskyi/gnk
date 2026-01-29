# HubSpot Form Integration Guide

A comprehensive guide for integrating HubSpot forms into Angular projects with pre-populated field support.

## Table of Contents

1. [Overview](#overview)
2. [HubSpot Setup](#hubspot-setup)
3. [Environment Configuration](#environment-configuration)
4. [Angular Component Implementation](#angular-component-implementation)
5. [Pre-populating Form Fields](#pre-populating-form-fields)
6. [Styling the Modal](#styling-the-modal)
7. [Troubleshooting](#troubleshooting)

---

## Overview

HubSpot forms can be embedded in Angular applications using their JavaScript API. The forms render inside an iframe, which creates some limitations but also provides isolation and consistent styling.

**Key Concepts:**

- HubSpot forms load via their `v2.js` script
- Forms render in a cross-origin iframe (cannot be manipulated directly via JavaScript)
- Pre-population works via URL query parameters (HubSpot reads these automatically)
- Each form has a Portal ID, Form ID, and Region

---

## HubSpot Setup

### 1. Create a Form in HubSpot

1. Go to **Marketing → Forms → Create form**
2. Add your fields (dropdowns, text inputs, etc.)
3. Note the **internal field names** (click on field → Advanced → Field name)
4. Save and publish the form

### 2. Get Form Credentials

After creating your form, you need:

| Credential | Where to Find                                        |
| ---------- | ---------------------------------------------------- |
| Portal ID  | Settings → Account Management → Account ID           |
| Form ID    | Forms → Select form → Embed → Look in the embed code |
| Region     | Usually `na1` (North America) or `eu1` (Europe)      |

Example embed code shows these values:

```html
<script>
  hbspt.forms.create({
    region: "eu1", // ← Region
    portalId: "147554099", // ← Portal ID
    formId: "78fd550b-...", // ← Form ID
  });
</script>
```

### 3. Configure Allowed Domains

To avoid 403 errors when embedding:

1. Go to **Settings → Marketing → Forms**
2. Find "Non-HubSpot Forms" or "Form sharing" section
3. Add your domains:
   - `localhost` (for development)
   - `yourdomain.com` (for production)

---

## Environment Configuration

Create environment files with HubSpot credentials:

### `environment.ts` (Development)

```typescript
export const environment = {
  production: false,
  apiUrl: "/api",
  hubspot: {
    portalId: "147554099", // Sandbox portal
    formId: "78fd550b-eec3-4958-bc4d-52c73924b87b",
    region: "eu1",
  },
};
```

### `environment.prod.ts` (Production)

```typescript
export const environment = {
  production: true,
  apiUrl: "/api",
  hubspot: {
    portalId: "4202168", // Production portal
    formId: "2d618652-614d-45f9-97fb-b9f88a6e8cc1",
    region: "eu1",
  },
};
```

### Configure File Replacement in `angular.json`

```json
{
  "configurations": {
    "production": {
      "fileReplacements": [
        {
          "replace": "src/environments/environment.ts",
          "with": "src/environments/environment.prod.ts"
        }
      ]
    }
  }
}
```

---

## Angular Component Implementation

### Full Modal Component

```typescript
import {
  Component,
  ChangeDetectionStrategy,
  input,
  output,
  effect,
  ElementRef,
  viewChild,
  PLATFORM_ID,
  inject,
} from "@angular/core";
import { isPlatformBrowser } from "@angular/common";
import { environment } from "../../../../environments/environment";

// Declare HubSpot global types
declare global {
  interface Window {
    hbspt?: {
      forms: {
        create: (config: HubSpotFormConfig) => void;
      };
    };
  }
}

interface HubSpotFormConfig {
  region: string;
  portalId: string;
  formId: string;
  target: string;
  onFormReady?: ($form: any) => void;
  onFormSubmitted?: () => void;
}

@Component({
  selector: "app-hubspot-form-modal",
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    @if (isOpen()) {
      <div
        class="fixed inset-0 z-[100] flex items-center justify-center p-4"
        (click)="onBackdropClick($event)"
      >
        <!-- Backdrop -->
        <div class="absolute inset-0 bg-black/70 backdrop-blur-sm"></div>

        <!-- Modal -->
        <div
          class="relative w-full max-w-2xl bg-[#18181b] rounded-2xl shadow-2xl overflow-hidden"
          (click)="$event.stopPropagation()"
        >
          <!-- Header -->
          <div
            class="flex items-center justify-between px-6 py-4 border-b border-gray-700"
          >
            <div>
              <h2 class="text-xl font-semibold text-white">{{ title() }}</h2>
              @if (subtitle()) {
                <p class="text-sm text-zinc-400 mt-0.5">{{ subtitle() }}</p>
              }
            </div>
            <button
              type="button"
              class="p-2 text-zinc-400 hover:text-white hover:bg-zinc-800 rounded-lg transition-colors"
              (click)="close.emit()"
              aria-label="Close modal"
            >
              <svg
                class="w-5 h-5"
                fill="none"
                stroke="currentColor"
                viewBox="0 0 24 24"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="2"
                  d="M6 18L18 6M6 6l12 12"
                />
              </svg>
            </button>
          </div>

          <!-- Form Container -->
          <div class="px-6 py-6 max-h-[70vh] overflow-y-auto bg-white">
            <div #formContainer class="hubspot-form-container"></div>
          </div>
        </div>
      </div>
    }
  `,
  styles: [
    `
      :host {
        display: contents;
      }

      :host ::ng-deep .hubspot-form-container {
        min-height: 200px;
      }
    `,
  ],
})
export class HubspotFormModalComponent {
  private readonly platformId = inject(PLATFORM_ID);

  // Inputs
  public readonly isOpen = input<boolean>(false);
  public readonly title = input<string>("Contact Us");
  public readonly subtitle = input<string | null>(null);
  public readonly prefilledValues = input<Record<string, string>>({});

  // Outputs
  public readonly close = output<void>();
  public readonly formSubmitted = output<void>();

  private readonly formContainer =
    viewChild<ElementRef<HTMLDivElement>>("formContainer");
  private formLoaded = false;
  private scriptLoaded = false;
  private originalUrl: string | null = null;

  constructor() {
    effect(() => {
      if (this.isOpen() && isPlatformBrowser(this.platformId)) {
        this.addUrlParams();
        setTimeout(() => this.loadForm(), 50);
      } else if (!this.isOpen() && isPlatformBrowser(this.platformId)) {
        this.removeUrlParams();
      }
    });
  }

  protected onBackdropClick(event: Event): void {
    if (event.target === event.currentTarget) {
      this.close.emit();
    }
  }

  /**
   * Add prefilled values as URL query parameters
   * HubSpot automatically reads these when the form loads
   */
  private addUrlParams(): void {
    const values = this.prefilledValues();
    if (!values || Object.keys(values).length === 0) return;

    this.originalUrl = window.location.href;

    const url = new URL(window.location.href);
    for (const [key, value] of Object.entries(values)) {
      url.searchParams.set(key, value);
    }

    window.history.replaceState({}, "", url.toString());
    console.log("HubSpot: Added URL params:", values);
  }

  /**
   * Remove query parameters and restore original URL
   */
  private removeUrlParams(): void {
    if (!this.originalUrl) return;
    window.history.replaceState({}, "", this.originalUrl);
    this.originalUrl = null;
  }

  private loadForm(): void {
    if (!isPlatformBrowser(this.platformId)) return;

    const container = this.formContainer()?.nativeElement;
    if (!container) return;

    container.innerHTML = "";
    this.formLoaded = false;
    this.assignContainerId(container);

    if (!this.scriptLoaded) {
      this.loadScript().then(() => this.createForm(container));
    } else {
      this.createForm(container);
    }
  }

  private loadScript(): Promise<void> {
    return new Promise((resolve, reject) => {
      if (window.hbspt?.forms) {
        this.scriptLoaded = true;
        resolve();
        return;
      }

      const existingScript = document.querySelector(
        'script[src*="hsforms.net/forms/v2"]',
      );
      if (existingScript) {
        existingScript.addEventListener("load", () => {
          this.scriptLoaded = true;
          resolve();
        });
        return;
      }

      const script = document.createElement("script");
      const region = environment.hubspot.region;
      script.src =
        region === "eu1"
          ? "https://js-eu1.hsforms.net/forms/v2.js"
          : "https://js.hsforms.net/forms/v2.js";
      script.charset = "utf-8";
      script.async = true;

      script.onload = () => {
        const checkReady = (attempts = 0): void => {
          if (window.hbspt?.forms) {
            this.scriptLoaded = true;
            resolve();
          } else if (attempts < 30) {
            setTimeout(() => checkReady(attempts + 1), 100);
          } else {
            reject(new Error("HubSpot forms API not available"));
          }
        };
        checkReady();
      };

      script.onerror = (error) => reject(error);
      document.head.appendChild(script);
    });
  }

  private createForm(container: HTMLElement): void {
    if (this.formLoaded || !window.hbspt?.forms) return;

    window.hbspt.forms.create({
      region: environment.hubspot.region,
      portalId: environment.hubspot.portalId,
      formId: environment.hubspot.formId,
      target: `#${container.id}`,
      onFormReady: () => {
        console.log("HubSpot: Form ready");
      },
      onFormSubmitted: () => {
        this.formSubmitted.emit();
      },
    });

    this.formLoaded = true;
  }

  private assignContainerId(container: HTMLElement): string {
    const id = "hubspot-form-" + Math.random().toString(36).substring(2, 9);
    container.id = id;
    return id;
  }
}
```

---

## Pre-populating Form Fields

### How It Works

HubSpot forms automatically read URL query parameters that match field internal names. When you navigate to:

```
https://yoursite.com/?field_name=value&another_field=value2
```

HubSpot will pre-fill those fields when the form loads.

### Finding Field Internal Names

1. In HubSpot, edit your form
2. Click on a field
3. Go to **Advanced** tab
4. Note the **Internal name** (e.g., `form_gonka_preffered_configuration`)

### Using in Component

```typescript
// In your parent component
protected openModal(gpuType: string): void {
  this.selectedGpuType.set(gpuType);
  this.prefilledValues.set({
    'form_gonka_preffered_configuration': '8 x A100',
    'form_gonka_servers_number': '1',
    'email': 'user@example.com'
  });
  this.isModalOpen.set(true);
}
```

```html
<!-- In template -->
<app-hubspot-form-modal
  [isOpen]="isModalOpen()"
  [title]="'Rent GPUs'"
  [subtitle]="'Selected: ' + selectedGpuType()"
  [prefilledValues]="prefilledValues()"
  (close)="closeModal()"
  (formSubmitted)="onFormSubmitted()"
/>
```

### Value Mapping Example

If your dropdown has specific values, create a mapping function:

```typescript
private getHubSpotValue(displayValue: string): string {
  const mapping: Record<string, string> = {
    '8x A100 Server': '8 x A100',
    '8x H100 Server': '8 x H100',
    '8x H200 Server': '8 x H200',
    '8x B200 Server': '8 x B200',
  };
  return mapping[displayValue] || displayValue;
}
```

---

## Styling the Modal

### Key CSS Considerations

1. **Dark header, white body**: HubSpot forms have a light theme by default
2. **Prevent white edges**: Use same background color for outer container and header
3. **Scrollable content**: Use `max-h-[70vh] overflow-y-auto` for long forms
4. **Proper width**: `max-w-2xl` allows two-column layouts in the form

### Custom Submit Button Styling

```css
:host ::ng-deep .hubspot-form-container {
  min-height: 200px;

  /* Style submit button */
  input[type="submit"],
  button[type="submit"],
  .hs-button {
    background: #ff4c00 !important;
    border-radius: 0.5rem !important;
    padding: 0.75rem 1.5rem !important;
  }

  input[type="submit"]:hover,
  button[type="submit"]:hover,
  .hs-button:hover {
    background: #e64500 !important;
  }
}
```

---

## Troubleshooting

### 403 Forbidden Errors

**Cause**: Domain not whitelisted in HubSpot settings.

**Solution**:

1. Go to HubSpot → Settings → Marketing → Forms
2. Add your domain to allowed list
3. For localhost, add `localhost` or `localhost:4200`

### Form Not Loading

**Cause**: Script loading issues or wrong region.

**Solution**:

```typescript
// Check region matches your HubSpot account location
script.src =
  region === "eu1"
    ? "https://js-eu1.hsforms.net/forms/v2.js" // Europe
    : "https://js.hsforms.net/forms/v2.js"; // North America
```

### Pre-filled Values Not Working

**Cause**: Field internal name mismatch or wrong value format.

**Solution**:

1. Double-check field internal name in HubSpot (Advanced tab)
2. Ensure dropdown values match exactly (including spaces)
3. Check browser console for "HubSpot: Added URL params" log

### Form Renders But Fields Are Empty

**Cause**: URL params added after form initialization.

**Solution**: Add URL params BEFORE loading the form:

```typescript
this.addUrlParams(); // First: add params
setTimeout(() => this.loadForm(), 50); // Then: load form
```

### Cross-Origin Iframe Restrictions

**Note**: You cannot directly manipulate form fields via JavaScript due to cross-origin iframe restrictions. The URL parameter approach is the only reliable method for pre-population.

---

## Quick Reference

### Script URLs by Region

| Region              | Script URL                               |
| ------------------- | ---------------------------------------- |
| North America (na1) | `https://js.hsforms.net/forms/v2.js`     |
| Europe (eu1)        | `https://js-eu1.hsforms.net/forms/v2.js` |

### Common Field Internal Names

These may vary by form - always check your specific form's field settings:

| Display Name | Possible Internal Name |
| ------------ | ---------------------- |
| Email        | `email`                |
| First Name   | `firstname`            |
| Last Name    | `lastname`             |
| Company      | `company`              |
| Phone        | `phone`                |

### Callbacks Available

```typescript
window.hbspt.forms.create({
  // ... config
  onFormReady: ($form) => {
    /* Form DOM ready */
  },
  onFormSubmit: ($form) => {
    /* Before submit */
  },
  onFormSubmitted: () => {
    /* After successful submit */
  },
});
```

---

## Example: Complete Implementation

See the MineGNK project for a complete working example:

- Component: `frontend/src/app/features/landing/components/hubspot-form-modal.component.ts`
- Environment: `frontend/src/environments/environment.ts`
- Usage: `frontend/src/app/features/landing/components/pricing-section.component.ts`
