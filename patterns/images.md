# Image Patterns

Upload, delete, reorder, and focal point handling for images in forms.

## Schema Types

Two new field types for entity schemas:

### Single Image (`image`)

Stores a Supabase Storage path as `text` column. Optional `focal_x`/`focal_y` columns for hero images.

```yaml
- name: titelbild
  type: image
  label: Titelbild
  required: true
  focal_point: true    # adds focal_x, focal_y columns
```

### Image Gallery (`images`)

Stored in a separate join table `{entity}_images` with columns for position, alt text, and focal point.

```yaml
- name: fotos
  type: images
  label: Fotos
  focal_point: true
```

Generated join table:
```sql
CREATE TABLE {entity}_images (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  {entity}_id UUID NOT NULL REFERENCES {entity}(id) ON DELETE CASCADE,
  storage_path TEXT NOT NULL,
  position INTEGER NOT NULL DEFAULT 0,
  alt_text TEXT,
  focal_x REAL NOT NULL DEFAULT 50,
  focal_y REAL NOT NULL DEFAULT 50,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_{entity}_images_{entity} ON {entity}_images({entity}_id);
```

## Supabase Storage Setup

Bucket config (run once, e.g. in migration or admin script):

```typescript
await supabase.storage.createBucket('images', {
  public: true,
  fileSizeLimit: '10MB',
  allowedMimeTypes: ['image/jpeg', 'image/png', 'image/webp'],
})
```

Upload utility:

```typescript
async function uploadImage(file: File, entityId: string): Promise<string> {
  const ext = file.name.split('.').pop()
  const path = `${entityId}/${crypto.randomUUID()}.${ext}`
  const { error } = await supabase.storage
    .from('images')
    .upload(path, file, { upsert: false, contentType: file.type })
  if (error) throw error
  return path
}
```

Public URL (with optional transform for thumbnails):

```typescript
function getImageUrl(path: string, transform?: { width: number; height: number }): string {
  const { data } = supabase.storage
    .from('images')
    .getPublicUrl(path, transform ? { transform } : undefined)
  return data.publicUrl
}
```

## Upload Flow

Upload happens **immediately on file select**, not on form save. The form field stores the resulting storage path string.

1. User selects or drops file
2. Client-side validation: type (jpg/png/webp), size (<10MB) via react-dropzone `accept` and `maxSize`
3. Local preview via `URL.createObjectURL(file)` shown immediately
4. File uploads to Supabase Storage
5. On success: storage path set into form field via `field.onChange(path)`, preview swaps to Supabase URL
6. On failure: toast error, dropzone shows error state, form field stays empty/previous value

### `ImageUploadField` (Single Image)

```tsx
import { useCallback, useState } from 'react'
import { useDropzone } from 'react-dropzone'
import { Loader2, X, ImageIcon } from 'lucide-react'
import { cn } from '@/lib/utils'
import { supabase } from '@/lib/supabase'
import { toast } from 'sonner'

interface ImageUploadFieldProps {
  value?: string | null
  onChange: (path: string | null) => void
  entityId: string
  className?: string
}

export function ImageUploadField({ value, onChange, entityId, className }: ImageUploadFieldProps) {
  const [uploading, setUploading] = useState(false)
  const [preview, setPreview] = useState<string | null>(null)

  const onDrop = useCallback(async (files: File[]) => {
    const file = files[0]
    if (!file) return

    const objectUrl = URL.createObjectURL(file)
    setPreview(objectUrl)
    setUploading(true)

    try {
      const ext = file.name.split('.').pop()
      const path = `${entityId}/${crypto.randomUUID()}.${ext}`
      const { error } = await supabase.storage
        .from('images')
        .upload(path, file, { contentType: file.type })
      if (error) throw error
      onChange(path)
      URL.revokeObjectURL(objectUrl)
      setPreview(null)
    } catch (err) {
      toast.error('Fehler beim Hochladen')
      URL.revokeObjectURL(objectUrl)
      setPreview(null)
    } finally {
      setUploading(false)
    }
  }, [entityId, onChange])

  const { getRootProps, getInputProps, isDragActive } = useDropzone({
    accept: { 'image/*': ['.jpg', '.jpeg', '.png', '.webp'] },
    maxSize: 10 * 1024 * 1024,
    maxFiles: 1,
    onDropAccepted: onDrop,
    onDropRejected: () => toast.error('Datei nicht erlaubt (max. 10MB, jpg/png/webp)'),
  })

  const imageUrl = preview ?? (value ? getImageUrl(value) : null)

  const handleRemove = (e: React.MouseEvent) => {
    e.stopPropagation()
    onChange(null)
  }

  return (
    <div
      {...getRootProps()}
      className={cn(
        'relative flex items-center justify-center rounded-md border-2 border-dashed cursor-pointer transition-colors',
        isDragActive ? 'border-primary bg-primary/5' : 'border-input hover:border-primary/50',
        imageUrl ? 'aspect-video' : 'aspect-video',
        className,
      )}
    >
      <input {...getInputProps()} />
      {uploading && (
        <div className="absolute inset-0 flex items-center justify-center bg-background/80 rounded-md">
          <Loader2 className="h-6 w-6 animate-spin text-muted-foreground" />
        </div>
      )}
      {imageUrl ? (
        <>
          <img src={imageUrl} alt="" className="h-full w-full object-cover rounded-md" />
          <button
            type="button"
            onClick={handleRemove}
            className="absolute top-2 right-2 rounded-full bg-background/80 p-1 hover:bg-background"
          >
            <X className="h-4 w-4" />
          </button>
        </>
      ) : (
        <div className="flex flex-col items-center gap-2 text-muted-foreground">
          <ImageIcon className="h-8 w-8" />
          <span className="text-sm">Bild hierher ziehen oder klicken</span>
        </div>
      )}
      <div aria-live="polite" className="sr-only">
        {uploading && 'Bild wird hochgeladen...'}
      </div>
    </div>
  )
}
```

## Delete Flow

- Removing an image from the form clears the field value immediately (UI feedback)
- Actual deletion from Supabase Storage happens on form save, inside the mutation
- No confirmation dialog — the unsaved changes guard protects against accidental navigation
- If user cancels the form, the image comes back (field value resets to saved state)

Storage cleanup in the update mutation:

```typescript
mutationFn: async (data) => {
  // Delete removed/replaced image from storage
  if (previousImagePath && previousImagePath !== data.image_path) {
    await supabase.storage.from('images').remove([previousImagePath])
  }
  const { error } = await supabase.from(table).update(data).eq('id', id)
  if (error) throw error
},
```

## Image Reordering (Gallery)

Use `@dnd-kit/core` + `@dnd-kit/sortable` for drag-and-drop reorder in a grid.

Dependencies: `react-dropzone`, `@dnd-kit/core`, `@dnd-kit/sortable`, `@dnd-kit/utilities`

### `SortableImageGrid`

```tsx
import { DndContext, closestCenter, KeyboardSensor, PointerSensor, useSensor, useSensors, type DragEndEvent } from '@dnd-kit/core'
import { SortableContext, rectSortingStrategy, useSortable, arrayMove } from '@dnd-kit/sortable'
import { CSS } from '@dnd-kit/utilities'
import { X, GripVertical } from 'lucide-react'

interface ImageItem {
  id: string
  storage_path: string
  position: number
  alt_text?: string | null
  focal_x: number
  focal_y: number
}

interface SortableImageGridProps {
  images: ImageItem[]
  onChange: (images: ImageItem[]) => void
  entityId: string
}

export function SortableImageGrid({ images, onChange, entityId }: SortableImageGridProps) {
  const sensors = useSensors(
    useSensor(PointerSensor, { activationConstraint: { distance: 5 } }),
    useSensor(KeyboardSensor),
  )

  function handleDragEnd(event: DragEndEvent) {
    const { active, over } = event
    if (!over || active.id === over.id) return
    const oldIndex = images.findIndex((i) => i.id === active.id)
    const newIndex = images.findIndex((i) => i.id === over.id)
    const reordered = arrayMove(images, oldIndex, newIndex).map((img, i) => ({ ...img, position: i }))
    onChange(reordered)
  }

  function handleRemove(id: string) {
    onChange(images.filter((i) => i.id !== id).map((img, i) => ({ ...img, position: i })))
  }

  return (
    <DndContext sensors={sensors} collisionDetection={closestCenter} onDragEnd={handleDragEnd}>
      <SortableContext items={images.map((i) => i.id)} strategy={rectSortingStrategy}>
        <div className="grid grid-cols-3 gap-3">
          {images.map((img) => (
            <SortableImageCard key={img.id} image={img} onRemove={() => handleRemove(img.id)} />
          ))}
        </div>
      </SortableContext>
    </DndContext>
  )
}

function SortableImageCard({ image, onRemove }: { image: ImageItem; onRemove: () => void }) {
  const { attributes, listeners, setNodeRef, transform, transition, isDragging } = useSortable({ id: image.id })

  const style = {
    transform: CSS.Transform.toString(transform),
    transition,
    opacity: isDragging ? 0.5 : 1,
  }

  return (
    <div ref={setNodeRef} style={style} className="group relative aspect-square rounded-md overflow-hidden border">
      <img
        src={getImageUrl(image.storage_path, { width: 300, height: 300 })}
        alt={image.alt_text ?? ''}
        className="h-full w-full object-cover"
        style={{ objectPosition: `${image.focal_x}% ${image.focal_y}%` }}
      />
      <div className="absolute top-1 left-1 opacity-0 group-hover:opacity-100 transition-opacity">
        <button type="button" {...attributes} {...listeners} className="rounded bg-background/80 p-1 cursor-grab">
          <GripVertical className="h-4 w-4" />
        </button>
      </div>
      <button
        type="button"
        onClick={onRemove}
        className="absolute top-1 right-1 rounded-full bg-background/80 p-1 opacity-0 group-hover:opacity-100 transition-opacity"
      >
        <X className="h-4 w-4" />
      </button>
    </div>
  )
}
```

dnd-kit handles touch (mobile), keyboard (Tab + Space/Enter + Arrows), and screen reader announcements natively.

## Focal Point

User clicks on the image to place a crosshair. Stored as `focal_x`/`focal_y` (0-100%, default 50/50). Applied via CSS `object-position`.

### `FocalPointPicker`

```tsx
import { useRef } from 'react'

interface FocalPointPickerProps {
  src: string
  value: { x: number; y: number }
  onChange: (point: { x: number; y: number }) => void
}

export function FocalPointPicker({ src, value, onChange }: FocalPointPickerProps) {
  const containerRef = useRef<HTMLDivElement>(null)

  const handleClick = (e: React.MouseEvent) => {
    const rect = containerRef.current!.getBoundingClientRect()
    const x = Math.round(((e.clientX - rect.left) / rect.width) * 100)
    const y = Math.round(((e.clientY - rect.top) / rect.height) * 100)
    onChange({
      x: Math.max(0, Math.min(100, x)),
      y: Math.max(0, Math.min(100, y)),
    })
  }

  return (
    <div className="space-y-3">
      <div ref={containerRef} className="relative cursor-crosshair rounded-md overflow-hidden" onClick={handleClick}>
        <img src={src} alt="" className="w-full" />
        <div
          className="absolute w-6 h-6 -translate-x-1/2 -translate-y-1/2 rounded-full border-2 border-white shadow-lg pointer-events-none"
          style={{ left: `${value.x}%`, top: `${value.y}%` }}
        >
          <div className="absolute inset-1 rounded-full bg-primary" />
        </div>
      </div>
      {/* Preview at different aspect ratios */}
      <div className="flex gap-3">
        <div className="space-y-1">
          <span className="text-xs text-muted-foreground">16:9</span>
          <div className="w-32 aspect-[16/9] overflow-hidden rounded border">
            <img src={src} alt="" className="h-full w-full object-cover" style={{ objectPosition: `${value.x}% ${value.y}%` }} />
          </div>
        </div>
        <div className="space-y-1">
          <span className="text-xs text-muted-foreground">1:1</span>
          <div className="w-20 aspect-square overflow-hidden rounded border">
            <img src={src} alt="" className="h-full w-full object-cover" style={{ objectPosition: `${value.x}% ${value.y}%` }} />
          </div>
        </div>
        <div className="space-y-1">
          <span className="text-xs text-muted-foreground">3:4</span>
          <div className="w-16 aspect-[3/4] overflow-hidden rounded border">
            <img src={src} alt="" className="h-full w-full object-cover" style={{ objectPosition: `${value.x}% ${value.y}%` }} />
          </div>
        </div>
      </div>
    </div>
  )
}
```

## Applying the Focal Point

Wherever the image is displayed (lists, cards, hero banners), apply the focal point:

```tsx
<img
  src={getImageUrl(path)}
  alt={altText}
  className="w-full h-64 object-cover"
  style={{ objectPosition: `${focalX}% ${focalY}%` }}
/>
```

Default (50/50) centers the image — same as `object-position: center`, so it's backwards compatible with images that have no focal point set.

## Alt Text

Every image field includes an `alt_text` input rendered below the preview. For the Zod schema:

```typescript
// Single image with alt text
titelbild: z.string().min(1, 'Bild ist erforderlich'),
titelbild_alt: z.string().optional().nullable(),

// Gallery images have alt_text per item in the join table
```

The alt text input is a standard `<Input>` with label "Alternativtext" below each image.

## Zod Schema Extension

Add to `generateZodSchema`:

```typescript
case 'image':
  schema = z.string() // storage path
  break
// Note: focal_x, focal_y, alt_text are separate fields generated automatically
// when focal_point: true is set in the schema

// 'images' type is handled via the join table, not as a direct form field
```

## Dependencies

```
react-dropzone @dnd-kit/core @dnd-kit/sortable @dnd-kit/utilities
```

---

## See also

- `patterns/form.md` — form layout where image fields are rendered
- `schema-format.md` — field type definitions
- `patterns/migration.md` — SQL generation including join tables for galleries
