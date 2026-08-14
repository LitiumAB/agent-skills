# Data Modelling in Litium

Reference for creating and working with fields and field templates in Litium.

## Overview

Litium uses a **Field Framework** to define data structures:
- **Field definition** — a typed data field (e.g., `SystemFieldTypeConstants.Text`, `SystemFieldTypeConstants.MultiField`)
- **Field template** — a named schema that groups field definitions and is assigned to entities (products, pages, blocks, media folders, etc.)

Docs: https://docs.litium.dev/platform/guides/data-modelling

---

## Field Templates

### Create a Folder Field Template (Media)

Docs: https://docs.litium.dev/platform/guides/data-modelling-how-to-create-a-folder-field-template

```csharp
using Litium.Media;
using Litium.Globalization;
using Litium.Runtime.DependencyInjection;

[Service(ServiceType = typeof(MediaFolderSetup), Lifetime = DependencyLifetime.Singleton)]
public class MediaFolderSetup
{
    private readonly FieldTemplateService _fieldTemplateService;
    private readonly LanguageService _languageService;
    private readonly FolderService _folderService;

    public MediaFolderSetup(FieldTemplateService fieldTemplateService,
        LanguageService languageService, FolderService folderService)
    {
        _fieldTemplateService = fieldTemplateService;
        _languageService = languageService;
        _folderService = folderService;
    }

    public FieldTemplate GetOrCreateFolderFieldTemplate(string id)
    {
        var template = _fieldTemplateService.Get<FolderFieldTemplate>(id);
        if (template == null)
        {
            template = new FolderFieldTemplate(id);
            foreach (var language in _languageService.GetAll())
                template.Localizations[language.CultureInfo].Name = id;
            _fieldTemplateService.Create(template);
        }
        return template;
    }

    public Folder GetOrCreateFolder(string id, Guid folderTemplateSystemId)
    {
        var folder = _folderService.Get(id);
        if (folder == null)
        {
            folder = new Folder(folderTemplateSystemId, id) { Id = id };
            _folderService.Create(folder);
        }
        return folder;
    }
}
```

### Create a Page Field Template

```csharp
var template = new PageFieldTemplate("MyPageTemplate")
{
    Localizations =
    {
        ["sv-SE"] = { Name = "Min sidmall" },
        ["en-US"] = { Name = "My page template" }
    },
    FieldGroups =
    {
        new FieldTemplateFieldGroup
        {
            Id = "Content",
            Collapsed = false,
            Fields = { "Title", "MainBody", "HeroImage" }
        }
    }
};
_fieldTemplateService.Create(template);
```

### Create a Block Field Template

```csharp
var template = new BlockFieldTemplate("MyBlockTemplate")
{
    Localizations =
    {
        ["sv-SE"] = { Name = "Mitt blockmall" },
        ["en-US"] = { Name = "My block template" }
    },
    FieldGroups =
    {
        new FieldTemplateFieldGroup
        {
            Id = "General",
            Collapsed = false,
            Fields = { "BlockTitle", "BlockText", "BlockImage" }
        }
    }
};
_fieldTemplateService.Create(template);
```

### Create a Product Field Template

```csharp
var template = new ProductFieldTemplate("MyProductTemplate")
{
    UseVariantUrl = true,
    Localizations =
    {
        ["sv-SE"] = { Name = "Min produktmall" },
        ["en-US"] = { Name = "My product template" }
    },
    ProductFieldGroups =
    {
        new FieldTemplateFieldGroup
        {
            Id = "General",
            Collapsed = false,
            Fields = { "MyField1", "MyField2" }
        }
    },
    VariantFieldGroups =
    {
        new FieldTemplateFieldGroup
        {
            Id = "Details",
            Fields = { "Color", "Size" }
        }
    }
};
_fieldTemplateService.Create(template);
```

---

## Multi-Field (Composite / Array Fields)

Use `SystemFieldTypeConstants.MultiField` to create composite fields that group multiple field types — useful for sliders, repeating content (e.g., image + link + text per item).

Docs: https://docs.litium.dev/platform/guides/data-modelling-how-to-set-up-a-multi-field

### Define and Create a Multi-Field

```csharp
var multiField = new FieldDefinition<CustomerArea>(
    "MyMultiField", SystemFieldTypeConstants.MultiField)
{
    Option = new MultiFieldOption
    {
        IsArray = true,
        Fields = new List<string>
        {
            SystemFieldDefinitionConstants.Name,
            SystemFieldDefinitionConstants.Email
        }
    }
};
_fieldDefinitionService.Create(multiField);
```

### Read and Write Multi-Field Values

```csharp
// Write
var item1 = new MultiFieldItem { AreaType = typeof(CustomerArea) };
item1.Fields.AddOrUpdateValue(SystemFieldDefinitionConstants.Name, CultureInfo.CurrentCulture, "Alice");
item1.Fields.AddOrUpdateValue(SystemFieldDefinitionConstants.Email, "alice@example.com");
entity.Fields.AddOrUpdateValue("MyMultiField", new[] { item1 });

// Read
entity.Fields.TryGetValue("MyMultiField", out var raw);
if (raw is IList<MultiFieldItem> items)
{
    var name = items[0].Fields.GetValue<string>(SystemFieldDefinitionConstants.Name, CultureInfo.CurrentCulture);
    var email = items[0].Fields.GetValue<string>(SystemFieldDefinitionConstants.Email);
}
```

### Custom Validation for Multi-Field

```csharp
[Service(Lifetime = DependencyLifetime.Scoped)]
internal class MultiFieldValidation : ValidationRuleBase<BaseProduct>
{
    public override ValidationResult Validate(BaseProduct entity, ValidationMode validationMode)
    {
        var result = new ValidationResult();
        var items = entity.Fields.GetValue<List<MultiFieldItem>>("myMultiField");
        for (int i = 0; i < items.Count; i++)
        {
            // Error key format: {multiFieldId}_{entitySystemId}|{fieldId}|{index+1}
            result.AddError($"myMultiField_{entity.SystemId}|Color|{i + 1}", "Color is required");
        }
        return result;
    }
}
```

---

## Custom Field Types

Use custom field types when the built-in types (Text, Int, Decimal, MediaPointer, etc.) are insufficient.

Docs: https://docs.litium.dev/platform/guides/data-modelling-how-to-create-custom-field-types

### Steps to Create a Custom Field Type

> **Version note:** The Angular Module Federation approach below applies to Litium versions using Angular-based back office extensions. For newer versions using the Web Component / Custom Element extension model, see `references/extensions/core-rules.md` and the matching framework reference instead.

1. **Define field type metadata** (`FieldTypeMetadataBase`)
2. **Implement edit field type converter** (`IEditFieldTypeConverter`)
3. **Optionally implement Excel converter** (`IExcelFieldTypeConverter`)
4. **Create Angular edit component** (inherits `BaseFieldEditor`)
5. **Register component in Angular module** (`extensions.ts`)
6. **Expose component in Webpack** (`ModuleFederationPlugin`)
7. **Register assembly as Angular module** (`AngularModuleAttribute`)

### 1. Field Type Metadata

```csharp
public class CustomTextFieldTypeMetadata : FieldTypeMetadataBase
{
    public override string Id => "CustomTextField";
    public override bool CanBeGridColumn => true;
    public override bool CanBeGridFilter => false;
    public override bool CanSort => false;
    public override Type JsonType => typeof(string);

    public override IFieldType CreateInstance(IFieldDefinition fieldDefinition)
    {
        var item = new CustomTextFieldType();
        item.Init(fieldDefinition);
        return item;
    }

    public class CustomTextFieldType : FieldTypeBase
    {
        public override object GetValue(ICollection<FieldData> fieldDatas)
            => fieldDatas.FirstOrDefault()?.TextValue;

        public override ICollection<FieldData> PersistFieldData(object item)
            => PersistFieldDataInternal(item);

        protected override ICollection<FieldData> PersistFieldDataInternal(object item)
            => new[] { new FieldData { TextValue = (string)item } };
    }
}
```

### 2. Edit Field Type Converter

```csharp
[Service(Name = "CustomTextField")]
internal class CustomTextEditFieldTypeConverter : IEditFieldTypeConverter
{
    private readonly IFieldTypeMetadata _fieldTypeMetadata;

    public CustomTextEditFieldTypeConverter(FieldTypeMetadataService svc)
        => _fieldTypeMetadata = svc.Get("CustomTextField");

    public object CreateOptionsModel() => null;

    public object ConvertFromEditValue(EditFieldTypeConverterArgs args, JToken item)
    {
        var instance = _fieldTypeMetadata.CreateInstance(args.FieldDefinition);
        return instance.ConvertFromJsonValue(item.ToObject(_fieldTypeMetadata.JsonType));
    }

    public JToken ConvertToEditValue(EditFieldTypeConverterArgs args, object item)
    {
        var instance = _fieldTypeMetadata.CreateInstance(args.FieldDefinition);
        var value = instance.ConvertToJsonValue(item);
        return value == null ? JValue.CreateNull() : JToken.FromObject(value);
    }

    // Module name (e.g. "Accelerator") # Component name
    public string EditComponentName => "Accelerator#FieldEditorCustomText";
    public string SettingsComponentName => string.Empty;
}
```

### 3. Angular Edit Component

```typescript
// field-editor-custom-text.component.ts
import { Component, ChangeDetectorRef, ChangeDetectionStrategy } from '@angular/core';
import { BaseFieldEditor } from 'litium-ui';

@Component({
  selector: 'field-editor-custom-text',
  templateUrl: './field-editor-custom-text.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class FieldEditorCustomText extends BaseFieldEditor {
  constructor(changeDetectorRef: ChangeDetectorRef) {
    super(changeDetectorRef);
  }
}
```

```html
<!-- field-editor-custom-text.component.html -->
<field-editor [field]="field">
  <p preview></p>
  <input edit type="text" [id]="name" #control
    [ngModel]="getValue(editLanguage)"
    (ngModelChange)="valueChange($event, editLanguage)" />
</field-editor>
```

### 4. AssemblyInfo / Module Registration

```csharp
// Properties/AssemblyInfo.cs
using Litium.Web.Administration;
[assembly: AngularModule("Accelerator")]
```

Build the Angular project after changes:
```powershell
cd Src/Litium.Accelerator.Administration.Extensions
yarn install
yarn run build
```

---

## Working with Fields Not in a Field Template

Fields can be set programmatically on entities even if they are not declared in a field template. This is done via the `Fields` collection:

Docs: https://docs.litium.dev/platform/guides/data-modelling-how-to-work-with-a-field-that-is-not-in-a-field-template

```csharp
// Setting a value on a field not in any template
entity.Fields.AddOrUpdateValue("MyStandaloneField", CultureInfo.CurrentCulture, "some value");
_entityService.Update(entity);

// Reading
var value = entity.Fields.GetValue<string>("MyStandaloneField", CultureInfo.CurrentCulture);
```

This is useful for system-level fields or fields set via import/integration that don't need to be editable in the back office UI.

---

## Built-in System Field Types

| Constant | Type |
|---|---|
| `SystemFieldTypeConstants.Text` | `string` (culture-dependent) |
| `SystemFieldTypeConstants.LimitedText` | `string` (short text, culture-dependent) |
| `SystemFieldTypeConstants.Int` | `int` |
| `SystemFieldTypeConstants.Decimal` | `decimal` |
| `SystemFieldTypeConstants.Boolean` | `bool` |
| `SystemFieldTypeConstants.Date` | `DateTimeOffset` |
| `SystemFieldTypeConstants.DateTime` | `DateTimeOffset` |
| `SystemFieldTypeConstants.MediaPointerFile` | `Guid` (file pointer) |
| `SystemFieldTypeConstants.MediaPointerImage` | `Guid` (image pointer) |
| `SystemFieldTypeConstants.Pointer` | `Guid` (generic pointer) |
| `SystemFieldTypeConstants.MultiField` | `List<MultiFieldItem>` |

> **Note:** Culture-dependent behavior is controlled by the `MultiCulture` property on a `FieldDefinition`, not by a separate field type.

---

## Useful Links

- Data modelling overview: https://docs.litium.dev/platform/guides/data-modelling
- Folder field template: https://docs.litium.dev/platform/guides/data-modelling-how-to-create-a-folder-field-template
- Multi-field: https://docs.litium.dev/platform/guides/data-modelling-how-to-set-up-a-multi-field
- Custom field types: https://docs.litium.dev/platform/guides/data-modelling-how-to-create-custom-field-types
- Field not in template: https://docs.litium.dev/platform/guides/data-modelling-how-to-work-with-a-field-that-is-not-in-a-field-template
