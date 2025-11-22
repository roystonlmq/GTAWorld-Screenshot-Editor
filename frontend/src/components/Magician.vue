// src/components/Magician.vue

<script setup lang="ts">
import { ref, computed, reactive, onMounted, watch } from 'vue';
import { type CSSProperties } from 'vue';
import Cookies from 'js-cookie';
import html2canvas from 'html2canvas';

interface ChatLayer {
  id: number;
  name: string;
  text: string;
  parsedLines: ParsedLine[];
  transform: { x: number; y: number; scale: number };
  visible: boolean;
}

const chatLayers = ref<ChatLayer[]>([]);
const selectedChatLayerId = ref<number | null>(null);
let nextChatLayerId = 1;

const createChatLayer = (options?: Partial<ChatLayer>) => {
  const layer: ChatLayer = {
    id: options?.id ?? nextChatLayerId++,
    name: options?.name || `Layer ${chatLayers.value.length + 1}`,
    text: options?.text || '',
    parsedLines: options?.parsedLines || [],
    transform: options?.transform || { x: 0, y: 0, scale: 1 },
    visible: options?.visible ?? true,
  };
  chatLayers.value.push(layer);
  return layer;
};

const activeChatLayer = computed(() => {
  if (selectedChatLayerId.value === null) return chatLayers.value[0];
  return chatLayers.value.find(layer => layer.id === selectedChatLayerId.value) || chatLayers.value[0];
});

const chatlogText = computed({
  get: () => activeChatLayer.value?.text || '',
  set: (val: string) => {
    if (activeChatLayer.value) {
      activeChatLayer.value.text = val;
    }
  }
});

// Bootstrap with a default layer
if (chatLayers.value.length === 0) {
  const initialLayer = createChatLayer({ name: 'Layer 1' });
  selectedChatLayerId.value = initialLayer.id;
}

const ensureActiveLayer = (): ChatLayer => {
  if (activeChatLayer.value) return activeChatLayer.value;
  const newLayer = createChatLayer({ name: `Layer ${chatLayers.value.length + 1}` });
  selectedChatLayerId.value = newLayer.id;
  return newLayer;
};
const droppedImageSrc = ref<string | null>(null); // To store the image data URL
const isDraggingOverDropZone = ref(false); // Renamed from isDragging for clarity
const dropZoneWidth = ref<number | null>(800); // Default width
const dropZoneHeight = ref<number | null>(600); // Default height
const fileInputRef = ref<HTMLInputElement | null>(null); // Ref for the file input element
const chatFileInputRef = ref<HTMLInputElement | null>(null); // New ref for chat file input
const showNewSessionDialog = ref(false); // Control dialog visibility
const stripTimestamps = ref(false); // New state for stripping timestamps
const screenshotTheme = ref(''); // Optional custom name for the saved screenshot
const alwaysPromptSaveLocation = ref(false); // If true, use the browser save dialog instead of auto-downloading
const filePickerSupported = computed(() => typeof window !== 'undefined' && 'showSaveFilePicker' in window);

// --- Resizable Chat Panel State ---
const chatPanelRef = ref<HTMLElement | null>(null);
const parentRowRef = ref<HTMLElement | null>(null);
const isResizingChatPanel = ref(false);
const resizeStartX = ref(0);
const initialChatPanelBasis = ref(25); // Default basis is 25%
const chatPanelFlexBasis = ref('25%'); // Default width as string with %
const mainContentFlexBasis = ref('75%'); // Default width for main content

// --- Parsed Chat State ---
interface ParsedLine {
  id: number;
  text: string;
  color?: string;
}
const parsedChatLines = computed<ParsedLine[]>({
  get: () => activeChatLayer.value?.parsedLines || [],
  set: (val: ParsedLine[]) => {
    if (activeChatLayer.value) {
      activeChatLayer.value.parsedLines = val;
    }
  }
});

// --- Image Manipulation State ---
const isImageDraggingEnabled = ref(false);
const imageTransform = reactive({ x: 0, y: 0, scale: 1 });
const isPanning = ref(false);
const panStart = reactive({ x: 0, y: 0 });
const panStartImagePos = reactive({ x: 0, y: 0 });

// --- Chat Manipulation State ---
const isChatDraggingEnabled = ref(false);
const chatTransform = reactive({ x: 0, y: 0, scale: 1 });
const isChatPanning = ref(false);
const chatPanStart = reactive({ x: 0, y: 0 });
const chatPanStartPos = reactive({ x: 0, y: 0 });

// Keep the working transform in sync with the active layer
watch(activeChatLayer, (layer) => {
  if (!layer) return;
  Object.assign(chatTransform, layer.transform);
}, { immediate: true });

watch(chatTransform, (val) => {
  if (activeChatLayer.value) {
    Object.assign(activeChatLayer.value.transform, val);
  }
}, { deep: true });

// --- Scale Adjustment for Drop Zone Visibility ---
const contentAreaRef = ref<HTMLElement | null>(null); // Reference to content area div
const dropzoneScale = ref(1); // Scale factor for the dropzone to fit screen
const isScaledDown = ref(false); // Flag to track if the dropzone is scaled down

// Add to the script setup section near the other state variables
const chatLineWidth = ref(640);

// Calculate necessary scale factor to fit dropzone in available viewport
const calculateDropzoneScale = () => {
  if (!contentAreaRef.value || !dropZoneWidth.value || !dropZoneHeight.value) return;
  
  // Get the available space in the content area (accounting for padding)
  const availableWidth = contentAreaRef.value.clientWidth - 24; // Adjust padding for better fit
  const availableHeight = contentAreaRef.value.clientHeight - 24;
  
  // Calculate the aspect ratio of the dropzone
  const dropzoneRatio = dropZoneWidth.value / dropZoneHeight.value;
  
  // First check if the dropzone would fit at 100% scale
  if (dropZoneWidth.value <= availableWidth && dropZoneHeight.value <= availableHeight) {
    // It fits without scaling
    dropzoneScale.value = 1;
    isScaledDown.value = false;
    return;
  }
  
  // Calculate scale needed for width and height
  const scaleX = availableWidth / dropZoneWidth.value;
  const scaleY = availableHeight / dropZoneHeight.value;
  
  // Use the smallest scale to ensure it fits in both dimensions while maintaining aspect ratio
  const newScale = Math.min(scaleX, scaleY);
  
  // Check if this would create awkward space
  const scaledWidth = dropZoneWidth.value * newScale;
  const scaledHeight = dropZoneHeight.value * newScale;
  
  // Calculate usage percentages
  const widthUsagePercent = (scaledWidth / availableWidth) * 100;
  const heightUsagePercent = (scaledHeight / availableHeight) * 100;
  
  // If we're using less than 75% of either dimension, see if we can optimize
  if (widthUsagePercent < 85 && heightUsagePercent < 85) {
    // Try to find a better scale that maximizes space usage
    const optimalScale = Math.min(
      availableWidth * 0.95 / dropZoneWidth.value,
      availableHeight * 0.95 / dropZoneHeight.value
    );
    
    if (optimalScale > newScale) {
      // We can use more space while keeping aspect ratio
      dropzoneScale.value = optimalScale;
      isScaledDown.value = true;
      console.log(`Optimized scaling: ${optimalScale.toFixed(2)}x to better use available space`);
      return;
    }
  }
  
  // Only update if there's a significant change to avoid unnecessary renders
  if (Math.abs(newScale - dropzoneScale.value) > 0.01) {
    dropzoneScale.value = newScale;
    isScaledDown.value = newScale < 0.98; // Consider it scaled down if below 98% of original size
    console.log(`Rescaled dropzone: ${newScale.toFixed(2)}x (${isScaledDown.value ? 'scaled down' : 'actual size'})`);
  }
};

// Add watchers to recalculate scale when dimensions change
watch([dropZoneWidth, dropZoneHeight, chatPanelFlexBasis], () => {
  // Recalculate scale whenever dropzone dimensions or chat panel width changes
  setTimeout(() => calculateDropzoneScale(), 0);
}, { immediate: true });

// Add a scale indicator to show when the content is scaled down
const scaleIndicator = computed(() => {
  if (!isScaledDown.value) return null;
  
  const scale = Math.round(dropzoneScale.value * 100);
  return `${scale}%`;
});

// Color mapping interface
interface ColorMapping {
  pattern: RegExp;
  color: string;
  splitPattern?: RegExp;
  markerColor?: string;
  fullLine?: boolean;
  checkPlayerName?: boolean;  // New flag
}

// Replace first/last name with single character name
const characterName = ref('');

// Watch character name changes and save to cookie
watch(characterName, (newValue) => {
  Cookies.set('characterName', newValue, { expires: 365 }); // Save for 1 year
});

// Update color mappings to handle all GTA World patterns
const colorMappings: ColorMapping[] = [
  // Radio messages - top priority
  { 
    pattern: /^\*\* \[S: .+? \| CH: .+?\]/i, 
    color: 'rgb(214, 207, 140)',
    fullLine: true
  },
  
  // Add cellphone pattern with higher priority
  { 
    pattern: /\(cellphone\)/i,
    color: 'rgb(251, 247, 36)',
    fullLine: true,
    checkPlayerName: true  // New flag to check if it's the player's message
  },
  
  // Basic chat patterns
  { pattern: /says:|shouts:/i, color: 'rgb(241, 241, 241)' },
  { pattern: /\(Car\)/i, color: 'rgb(251, 247, 36)' },
  { pattern: /^\*/, color: 'rgb(194, 163, 218)' },
  { pattern: /\bwhispers\b/i, color: 'rgb(237, 168, 65)' },
  { pattern: /\bYou paid\b|\bpaid you\b|\byou gave\b|\bgave you\b|\bYou received\b/i, color: 'rgb(86, 214, 75)' },
  { pattern: /g\)/i, color: 'rgb(255, 255, 0)' },
  { pattern: /\[low\]:|\[lower\]:/i, color: 'rgb(150, 149, 149)' },

  // New patterns from the provided code
  { 
    pattern: /\[!]/, 
    color: 'rgb(255, 255, 255)', 
    splitPattern: /(\[!])/, 
    markerColor: 'rgb(255, 0, 195)'
  },
  { pattern: /\[INFO]:/, color: 'rgb(255, 255, 255)', splitPattern: /(\[INFO]:)/, markerColor: 'rgb(27, 124, 222)' },
  { pattern: /\[ALERT]/, color: 'rgb(255, 255, 255)', splitPattern: /(\[ALERT])/, markerColor: 'rgb(27, 124, 222)' },
  { pattern: /\[GYM]/, color: 'rgb(255, 255, 255)', splitPattern: /(\[GYM])/, markerColor: 'rgb(22, 106, 189)' },
  { pattern: /\[advertisement]/i, color: 'rgb(127, 239, 43)' },
  { pattern: /\(\( \(PM/i, color: 'rgb(239, 227, 0)' },
  { pattern: /\(\( \(/i, color: 'rgb(139, 138, 138)' },
  { pattern: /\[megaphone]/i, color: 'rgb(241, 213, 3)' },
  { pattern: /\[microphone]/i, color: 'rgb(246, 218, 3)' },
  { pattern: /\[intercom]/i, color: 'rgb(26, 131, 232)' },
  
  // Character kill pattern
  { pattern: /\[Character kill]/, color: 'rgb(240, 0, 0)', splitPattern: /(\[Character kill])/, markerColor: 'rgb(56, 150, 243)' },
  
  // Money patterns
  { 
    pattern: /\[\$.*g\)/, 
    color: 'rgb(255, 255, 0)',
    splitPattern: /(\[\$[^\]]*\])/,
    markerColor: 'rgb(86, 214, 75)'
  },
  { 
    pattern: /\(\$.*g\)/, 
    color: 'rgb(255, 255, 0)',
    splitPattern: /(\(\$[^\)]*\))/,
    markerColor: 'rgb(86, 214, 75)'
  },
  
  // Phone number pattern
  { 
    pattern: / PH: .*g\)/, 
    color: 'rgb(255, 255, 0)',
    splitPattern: /( PH: .*)/,
    markerColor: 'rgb(86, 214, 75)'
  }
];

// Computed style for the drop zone
const dropZoneStyle = computed(() => {
  // Calculate scale if needed
  const scale = isScaledDown.value ? dropzoneScale.value : 1;
  
  return {
    width: dropZoneWidth.value ? `${dropZoneWidth.value}px` : '800px', 
    height: dropZoneHeight.value ? `${dropZoneHeight.value}px` : '600px',
    backgroundColor: '#e0e0e0',
    borderRadius: '4px',
    position: 'absolute',
    top: '50%',
    left: '50%',
    transform: `translate(-50%, -50%) scale(${scale})`,
    transformOrigin: 'center center',
    overflow: 'hidden',
    cursor: isImageDraggingEnabled.value ? (isPanning.value ? 'grabbing' : 'grab') : 'default',
    transition: 'transform 0.2s ease',
    border: isScaledDown.value ? '2px solid #42a5f5' : '2px dashed transparent' // Blue border when scaled
  };
});

// Aspect ratio container style - use the paddingTop technique to maintain ratio
const aspectRatioContainerStyle = computed(() => {
  const aspectRatio = dropZoneWidth.value && dropZoneHeight.value
    ? dropZoneHeight.value / dropZoneWidth.value
    : 0.5625; // Default 16:9 aspect ratio
    
  return {
    paddingTop: `${aspectRatio * 100}%`,
    width: '100%',
    position: 'relative' as const,
    maxHeight: 'calc(100% - 16px)',
    margin: '8px 0'
  };
});

const imageStyle = computed(() => ({
  maxWidth: 'none', // Allow image to be larger than container for zoom
  maxHeight: 'none',
  position: 'absolute', // Position relative to the drop zone sheet
  top: '0', // Start at top-left before transform
  left: '0',
  transformOrigin: 'center center', // Zoom from the center
  transform: `translate(${imageTransform.x}px, ${imageTransform.y}px) scale(${imageTransform.scale})`,
  cursor: isImageDraggingEnabled.value ? (isPanning.value ? 'grabbing' : 'grab') : 'default',
  transition: isPanning.value ? 'none' : 'transform 0.1s ease-out' // Smooth transition only when not panning
}));

const clearChatlog = () => {
  chatlogText.value = '';
  parsedChatLines.value = []; // Clear parsed lines too
};

const parseChatlog = (layer: ChatLayer | undefined = activeChatLayer.value) => {
  const targetLayer = layer ?? ensureActiveLayer();
  targetLayer.visible = true;
  const sourceText = targetLayer.text || '';
  if (!sourceText) {
    targetLayer.parsedLines = [];
    parsedChatLines.value = [];
    renderKey.value++;
    return;
  }

  const lines = sourceText.split('\n').filter(line => line.trim() !== '');
  const parsed = lines.map((line, index) => {
    let processedText = line;
    
    // Strip timestamps if the option is enabled
    if (stripTimestamps.value) {
      processedText = processedText.replace(/^\[\d{2}:\d{2}:\d{2}\]\s*/, '');
    }
    
    let color: string | undefined = undefined;
    
    // First check for cellphone messages
    const cellphonePattern = colorMappings.find(mapping => 
      mapping.checkPlayerName && mapping.pattern.test(processedText)
    );
    
    if (cellphonePattern && characterName.value) {
      // If it's the player's message, use white color
      if (processedText.startsWith(characterName.value)) {
        color = 'rgb(255, 255, 255)';
      } else {
        // Otherwise use yellow for incoming calls
        color = cellphonePattern.color;
      }
    } else {
      // Check for patterns that should color the entire line
      const fullLinePattern = colorMappings.find(mapping => 
        mapping.fullLine && !mapping.checkPlayerName && mapping.pattern.test(processedText)
      );

      if (fullLinePattern) {
        color = fullLinePattern.color;
      } else {
        // Check for split patterns 
        const splitPattern = colorMappings.find(mapping => 
          mapping.splitPattern && mapping.pattern.test(processedText)
        );

        if (splitPattern && splitPattern.splitPattern && splitPattern.markerColor) {
          const parts = processedText.split(splitPattern.splitPattern);
          if (parts.length > 1) {
            processedText = parts.map((part, i) => {
              if (splitPattern.splitPattern?.test(part)) {
                return `<span style="color: ${splitPattern.markerColor}">${part}</span>`;
              }
              return `<span style="color: ${splitPattern.color}">${part}</span>`;
            }).join('');
            color = 'white'; // Base color doesn't matter as we're using inline styles
          }
        } else {
          // Check other patterns
          for (const mapping of colorMappings) {
            if (mapping.pattern.test(processedText)) {
              color = mapping.color;
              break;
            }
          }
        }
      }
    }
    
    return {
      id: index,
      text: processedText,
      color: color || 'white'
    };
  });

  targetLayer.parsedLines = parsed;
  parsedChatLines.value = parsed;
  renderKey.value++;
};

const selectChatLayer = (id: number) => {
  const layer = chatLayers.value.find(l => l.id === id);
  if (!layer) {
    const fallback = ensureActiveLayer();
    selectedChatLayerId.value = fallback.id;
    parsedChatLines.value = fallback.parsedLines;
    chatlogText.value = fallback.text;
    Object.assign(chatTransform, fallback.transform);
    return;
  }
  selectedChatLayerId.value = id;
  if (layer.text && layer.parsedLines.length === 0) {
    parseChatlog(layer);
  }
  chatlogText.value = layer.text;
  parsedChatLines.value = layer.parsedLines;
  Object.assign(chatTransform, layer.transform);
  selectedText.lineIndex = -1;
  selectedText.layerId = -1;
};

const addChatLayer = () => {
  const layer = createChatLayer();
  selectChatLayer(layer.id);
};

const removeChatLayer = (id: number) => {
  if (chatLayers.value.length <= 1) return; // Always keep at least one layer
  const index = chatLayers.value.findIndex(l => l.id === id);
  if (index === -1) return;

  chatLayers.value.splice(index, 1);
  censoredRegions.value = censoredRegions.value.filter(region => region.layerId !== id);

  // Select the previous layer or the first one
  const fallback = chatLayers.value[index - 1] || chatLayers.value[0];
  selectChatLayer(fallback.id);
};

const moveChatLayer = (id: number, direction: 'up' | 'down') => {
  const index = chatLayers.value.findIndex(l => l.id === id);
  if (index === -1) return;
  const swapWith = direction === 'up' ? index - 1 : index + 1;
  if (swapWith < 0 || swapWith >= chatLayers.value.length) return;
  const layers = chatLayers.value;
  [layers[index], layers[swapWith]] = [layers[swapWith], layers[index]];
};

// --- File Handling Methods ---

// Trigger click on hidden file input when the import button is clicked
const triggerFileInput = () => {
  if (fileInputRef.value) {
    fileInputRef.value.click();
  }
};

// Handle image file selection from file picker
const handleFileSelect = (event: Event) => {
  const target = event.target as HTMLInputElement;
  const files = target.files;
  
  if (files && files.length > 0) {
    const file = files[0];
    
    // Basic image type validation
    if (file.type.startsWith('image/')) {
      const reader = new FileReader();
      reader.onload = (e) => {
        droppedImageSrc.value = e.target?.result as string;
        // Reset image transform
        imageTransform.x = 0;
        imageTransform.y = 0;
        imageTransform.scale = 1;
      };
      reader.readAsDataURL(file);
    } else {
      console.warn('Selected file is not an image.');
      // Optionally show an error message to the user
    }
    
    // Reset the input so the same file can be selected again
    target.value = '';
  }
};

// Handle chatlog file selection
const handleChatFileSelect = async (event: Event) => {
  const target = event.target as HTMLInputElement;
  const files = target.files;
  
  if (files && files.length > 0) {
    const file = files[0];
    
    // Basic text file validation
    if (file.type === 'text/plain' || file.name.endsWith('.txt')) {
      try {
        const text = await file.text(); // Read file as text
        chatlogText.value = text; // Set the textarea content
        parseChatlog(); // Automatically parse the imported chat
      } catch (error) {
        console.error('Error reading chat file:', error);
        // Optionally show an error message to the user
      }
    } else {
      console.warn('Selected file is not a text file.');
      // Optionally show an error message to the user
    }
    
    // Reset the input so the same file can be selected again
    target.value = '';
  }
};

// Trigger chat file input
const triggerChatFileInput = () => {
  if (chatFileInputRef.value) {
    chatFileInputRef.value.click();
  }
};

// --- Drag and Drop Handlers ---
const isImageFile = (file: File) => {
  if (file.type && file.type.startsWith('image/')) return true;
  return /\.(png|jpe?g|webp|gif|bmp|tiff)$/i.test(file.name);
};

const handleDragOver = (event: DragEvent) => {
  event.preventDefault(); // Prevent default behavior
  event.stopPropagation();
  if (event.dataTransfer) {
    event.dataTransfer.dropEffect = 'copy';
  }
  isDraggingOverDropZone.value = true;
};

const handleDragLeave = () => {
  isDraggingOverDropZone.value = false;
};

const handleDrop = (event: DragEvent) => {
  event.preventDefault();
  event.stopPropagation();
  isDraggingOverDropZone.value = false;

  const dt = event.dataTransfer;
  const file =
    dt?.files && dt.files.length > 0
      ? dt.files[0]
      : Array.from(dt?.items || [])
        .filter(item => item.kind === 'file')
        .map(item => item.getAsFile())
        .find(Boolean) || null;

  if (!file) {
    console.warn('No files detected on drop.');
    return;
  }
  if (!isImageFile(file)) {
    console.warn('Dropped file is not an image.');
    return;
  }

  const reader = new FileReader();
  reader.onload = (e) => {
    droppedImageSrc.value = e.target?.result as string;
    imageTransform.x = 0;
    imageTransform.y = 0;
    imageTransform.scale = 1;
  };
  reader.readAsDataURL(file);
};

// --- Image Panning Handlers ---
const handleImageMouseDown = (event: MouseEvent) => {
  if (!isImageDraggingEnabled.value || !droppedImageSrc.value) return;
  event.preventDefault();
  isPanning.value = true;
  panStart.x = event.clientX;
  panStart.y = event.clientY;
  panStartImagePos.x = imageTransform.x;
  panStartImagePos.y = imageTransform.y;
  // Add a class to body to prevent text selection during drag
  document.body.style.userSelect = 'none';
};

const handleImageMouseMove = (event: MouseEvent) => {
  if (!isPanning.value) return;
  const deltaX = event.clientX - panStart.x;
  const deltaY = event.clientY - panStart.y;
  imageTransform.x = panStartImagePos.x + deltaX;
  imageTransform.y = panStartImagePos.y + deltaY;
};

const handleImageMouseUpOrLeave = () => {
  if (isPanning.value) {
    isPanning.value = false;
    document.body.style.userSelect = ''; // Re-enable text selection
  }
};

// --- Image Zoom Handler ---
const handleWheel = (event: WheelEvent) => {
  if (!isImageDraggingEnabled.value || !droppedImageSrc.value) return;
  event.preventDefault();
  const scaleAmount = 0.1;
  const minScale = 0.1;
  const maxScale = 10;

  if (event.deltaY < 0) {
    // Zoom in
    imageTransform.scale = Math.min(maxScale, imageTransform.scale + scaleAmount);
  } else {
    // Zoom out
    imageTransform.scale = Math.max(minScale, imageTransform.scale - scaleAmount);
  }
};

// --- Toolbar Button Handlers ---
const toggleImageDrag = () => {
  isImageDraggingEnabled.value = !isImageDraggingEnabled.value;
  if (isImageDraggingEnabled.value && isChatDraggingEnabled.value) {
    isChatDraggingEnabled.value = false; // Disable chat drag if enabling image drag
  }
};

// --- Chat Dragging Handlers ---
const handleChatMouseDown = (event: MouseEvent) => {
  if (!isChatDraggingEnabled.value || parsedChatLines.value.length === 0) return;
  
  // Don't allow chat dragging if image dragging is active and in progress
  if (isImageDraggingEnabled.value && isPanning.value) {
    event.stopPropagation();
    return;
  }
  
  event.preventDefault();
  event.stopPropagation(); // Stop event from propagating to image
  isChatPanning.value = true;
  chatPanStart.x = event.clientX;
  chatPanStart.y = event.clientY;
  chatPanStartPos.x = chatTransform.x;
  chatPanStartPos.y = chatTransform.y;
  document.body.style.userSelect = 'none';
};

const handleChatMouseMove = (event: MouseEvent) => {
  if (!isChatPanning.value) return;
  
  // Prevent event from being handled by image drag
  event.stopPropagation();
  
  const deltaX = event.clientX - chatPanStart.x;
  const deltaY = event.clientY - chatPanStart.y;
  chatTransform.x = chatPanStartPos.x + deltaX;
  chatTransform.y = chatPanStartPos.y + deltaY;
};

const handleChatMouseUpOrLeave = (event?: MouseEvent) => {
  if (isChatPanning.value) {
    if (event) {
      event.stopPropagation();
    }
    isChatPanning.value = false;
    document.body.style.userSelect = '';
  }
};

// --- Chat Zoom Handler ---
const handleChatWheel = (event: WheelEvent) => {
  if (!isChatDraggingEnabled.value || parsedChatLines.value.length === 0) return;
  event.preventDefault();
  const scaleAmount = 0.1;
  const minScale = 0.5;
  const maxScale = 3;

  if (event.deltaY < 0) {
    chatTransform.scale = Math.min(maxScale, chatTransform.scale + scaleAmount);
  } else {
    chatTransform.scale = Math.max(minScale, chatTransform.scale - scaleAmount);
  }
};

// Toggle chat dragging
const toggleChatDrag = () => {
  isChatDraggingEnabled.value = !isChatDraggingEnabled.value;
  if (isChatDraggingEnabled.value && isImageDraggingEnabled.value) {
    isImageDraggingEnabled.value = false; // Disable image drag if enabling chat drag
  }
};

// --- Chat Panel Resize Handlers ---
const handleResizeMouseDown = (event: MouseEvent) => {
  // Prevent default behavior (like text selection)
  event.preventDefault();
  event.stopPropagation();
  
  console.log("Starting resize...");
  
  // Record starting coordinates and initial width
  resizeStartX.value = event.clientX;
  
  // Store current width percentage
  const currentBasis = parseFloat(chatPanelFlexBasis.value);
  initialChatPanelBasis.value = isNaN(currentBasis) ? 25 : currentBasis;
  
  // Set flag for resizing state
  isResizingChatPanel.value = true;
  
  // Set cursor and user-select for entire document during resize
  document.body.style.cursor = 'col-resize';
  document.body.style.userSelect = 'none';
  
  // Add event listeners for move and up directly to document
  document.addEventListener('mousemove', handleResizeMouseMove);
  document.addEventListener('mouseup', handleResizeMouseUp);
};

const handleResizeMouseMove = (event: MouseEvent) => {
  // Only proceed if we're in resizing state
  if (!isResizingChatPanel.value) return;
  
  // Prevent default behavior
  event.preventDefault();
  event.stopPropagation();
  
  // Get the movement delta - positive when moving left (making panel wider)
  const deltaX = resizeStartX.value - event.clientX;
  
  // Calculate direct conversion to percentage - using parent container width
  const parentWidth = parentRowRef.value?.offsetWidth || window.innerWidth;
  const deltaPercent = (deltaX / parentWidth) * 100;
  
  // Calculate new basis value - original percentage + delta
  let newBasis = initialChatPanelBasis.value + deltaPercent;
  
  // Calculate minimum width based on drop zone size to prevent panel from disappearing
  // As drop zone gets wider, we need to ensure chat panel has a reasonable minimum width
  const minWidthPx = 250; // Absolute minimum width in pixels
  let minWidthPercent = 15; // Default minimum percentage
  
  if (dropZoneWidth.value && dropZoneWidth.value > 800) {
    // For wider drop zones, calculate a higher minimum percentage
    // This ensures the chat panel stays visible even with wide drop zones
    const minPercentForCurrentWidth = (minWidthPx / parentWidth) * 100;
    minWidthPercent = Math.max(minWidthPercent, minPercentForCurrentWidth);
    
    console.log(`Adjusted min width: ${minWidthPercent.toFixed(2)}% (${minWidthPx}px)`);
  }
  
  // Clamp the value to reasonable boundaries
  // The maximum is still 50% but minimum is now dynamic based on drop zone width
  newBasis = Math.max(minWidthPercent, Math.min(newBasis, 50));
  
  // Apply the new width percentage
  chatPanelFlexBasis.value = `${newBasis}%`;
  mainContentFlexBasis.value = `${100 - newBasis}%`;
  
  console.log(`Resizing: delta=${deltaX.toFixed(0)}px (${deltaPercent.toFixed(2)}%), new width=${newBasis.toFixed(2)}%`);
};

const handleResizeMouseUp = () => {
  // Clean up - reset flags and styles
  isResizingChatPanel.value = false;
  document.body.style.cursor = '';
  document.body.style.userSelect = '';
  
  // Remove the event listeners
  document.removeEventListener('mousemove', handleResizeMouseMove);
  document.removeEventListener('mouseup', handleResizeMouseUp);
  
  console.log("Resize complete");
};

// Replace chatLineStyle with a direct rendering approach
const chatLineStyle = computed(() => (line: ParsedLine, index: number) => {
  // Hard code basic positioning
  return {
    position: 'relative' as const,
    top: 0,
    left: 0,
    marginBottom: '0',
    padding: 0,
    boxSizing: 'border-box' as const,
  };
});

// Update chat overlay styles
const chatStyles = computed(() => {
  return (layer: ChatLayer) => ({
    position: 'absolute' as const,
    top: '0',
    left: '0',
    width: '100%',
    height: 'auto',
    maxWidth: `${dropZoneWidth.value}px`, // Match the exact dropzone width
    fontFamily: '"Arial Black", Arial, sans-serif',
    fontSize: '12px',
    lineHeight: '16px',
    transform: `translate(${layer.transform.x}px, ${layer.transform.y}px) scale(${layer.transform.scale})`,
    transformOrigin: 'top left',
    pointerEvents: (isChatDraggingEnabled.value && layer.id === activeChatLayer.value?.id ? 'auto' : 'none') as 'auto' | 'none',
    wordWrap: 'break-word' as const,
    whiteSpace: 'pre-wrap' as const,
    opacity: layer.visible ? 1 : 0.25,
    filter: layer.visible ? 'none' : 'grayscale(1)'
  });
});

const saveBlobWithFilePicker = async (blob: Blob, filename: string) => {
  const picker = (window as any).showSaveFilePicker;
  if (!picker) {
    throw new Error('File picker not supported');
  }

  const handle = await picker({
    suggestedName: filename,
    types: [
      {
        description: 'PNG Image',
        accept: { 'image/png': ['.png'] }
      }
    ]
  });

  const writable = await (handle as any).createWritable();
  await writable.write(blob);
  await writable.close();
};

const downloadBlob = (blob: Blob, filename: string) => {
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = filename;
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
};

const saveBlob = async (blob: Blob, filename: string) => {
  if (alwaysPromptSaveLocation.value && filePickerSupported.value) {
    try {
      await saveBlobWithFilePicker(blob, filename);
      return;
    } catch (error) {
      console.warn('Falling back to default download:', error);
    }
  }

  downloadBlob(blob, filename);
};

const buildScreenshotFilename = () => {
  const now = new Date();
  const pad = (value: number) => value.toString().padStart(2, '0');
  const datePart = `${pad(now.getDate())}${pad(now.getMonth() + 1)}${now.getFullYear()}`;
  const timePart = `${pad(now.getHours())}${pad(now.getMinutes())}`;
  const rawTheme = screenshotTheme.value.trim();
  const safeTheme = rawTheme
    .replace(/[^a-zA-Z0-9\s_-]/g, '')
    .trim()
    .replace(/\s+/g, '-');
  const finalTheme = safeTheme || 'screenshot';
  return `${datePart} - ${timePart} - ${finalTheme}.png`;
};

// Update the saveImage function to ensure 1:1 positioning match with preview
const saveImage = async () => {
  if (!droppedImageSrc.value) {
    console.warn('No image to save');
    return;
  }

  try {
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');
    if (!ctx) {
      throw new Error('Could not get canvas context');
    }

    canvas.width = dropZoneWidth.value || 800;
    canvas.height = dropZoneHeight.value || 600;

    ctx.fillStyle = '#000000';
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    ctx.save();
    const img = new Image();
    img.src = droppedImageSrc.value;
    await new Promise((resolve, reject) => {
      img.onload = resolve;
      img.onerror = reject;
    });

    const viewportRatio = canvas.width / canvas.height;
    const imageRatio = img.naturalWidth / img.naturalHeight;
    let drawWidth, drawHeight, offsetX, offsetY;

    if (imageRatio > viewportRatio) {
      drawWidth = canvas.width;
      drawHeight = canvas.width / imageRatio;
      offsetX = 0;
      offsetY = (canvas.height - drawHeight) / 2;
    } else {
      drawHeight = canvas.height;
      drawWidth = canvas.height * imageRatio;
      offsetX = (canvas.width - drawWidth) / 2;
      offsetY = 0;
    }

    ctx.translate(offsetX + drawWidth / 2, offsetY + drawHeight / 2);
    ctx.scale(imageTransform.scale, imageTransform.scale);
    ctx.translate(imageTransform.x, imageTransform.y);
    ctx.drawImage(img, -drawWidth / 2, -drawHeight / 2, drawWidth, drawHeight);
    ctx.restore();

    const visibleChatLayers = chatLayers.value.filter(layer => layer.visible && layer.parsedLines.length > 0);
    for (const layer of visibleChatLayers) {
      ctx.save();
      ctx.translate(layer.transform.x, layer.transform.y);
      ctx.scale(layer.transform.scale, layer.transform.scale);

      ctx.font = '700 12px Arial, sans-serif';
      ctx.textBaseline = 'top';
      ctx.textRendering = 'geometricPrecision';
      ctx.letterSpacing = '0px';
      let currentY = 0;
      const getTextColor = (text: string): string => {
        if (text.includes('[S:') && text.includes('CH:')) return 'rgb(214, 207, 140)';
        if (text.startsWith('*')) return 'rgb(194, 163, 218)';
        if (text.includes('(Car)') && text.includes('whispers:')) return 'rgb(255, 255, 0)';
        if (text.includes('whispers:')) return 'rgb(237, 168, 65)';
        if (text.includes('(cellphone)')) {
          if (text.startsWith(characterName.value)) {
            return 'rgb(255, 255, 255)';
          }
          return 'rgb(251, 247, 36)';
        }
        if (text.includes('[Megaphone]')) return 'rgb(241, 213, 3)';
        if (text.includes('You paid')) return 'rgb(86, 214, 75)';
        return 'rgb(255, 255, 255)';
      };

      const drawBlackBar = (y: number, width: number) => {
        if (showBlackBars.value) {
          ctx.fillStyle = '#000000';
          ctx.fillRect(0, y, width + 8, 17);
        }
      };

      const drawTextWithOutline = (text: string, x: number, y: number, color: string, censorType?: CensorType) => {
        const width = ctx.measureText(text).width;
        const TEXT_OFFSET_Y = 1;

        if (showBlackBars.value) {
          ctx.fillStyle = '#000000';
          ctx.fillRect(x - 2, y, width + 4, 16);
        }

        if (censorType) {
          const tempCanvas = document.createElement('canvas');
          const tempCtx = tempCanvas.getContext('2d');
          if (!tempCtx) return;

          tempCanvas.width = width + 20;
          tempCanvas.height = 16 + 20;

          tempCtx.font = ctx.font;
          tempCtx.fillStyle = color;
          tempCtx.fillText(text, 10, 10 + TEXT_OFFSET_Y);

          tempCtx.globalCompositeOperation = 'source-in';
          tempCtx.filter = 'blur(4px)';
          tempCtx.fillStyle = 'black';
          tempCtx.fillRect(0, 0, tempCanvas.width, tempCanvas.height);
          tempCtx.filter = 'none';
          tempCtx.globalCompositeOperation = 'source-over';

          ctx.drawImage(tempCanvas, x - 10, y - 10);
          return;
        }

        const shadowOffsets = [
          [-1, -1], [-1, 0], [-1, 1],
          [0, -1],           [0, 1],
          [1, -1], [1, 0], [1, 1]
        ];
        ctx.fillStyle = '#000000';
        shadowOffsets.forEach(([dx, dy]) => {
          ctx.fillText(text, x + dx, y + TEXT_OFFSET_Y + dy);
        });

        ctx.fillStyle = color;
        ctx.fillText(text, x, y + TEXT_OFFSET_Y);
      };

      const getCensorType = (lineIndex: number, startPos: number, length: number, layerId: number): CensorType | undefined => {
        const region = censoredRegions.value.find(r =>
          r.layerId === layerId &&
          r.lineIndex === lineIndex &&
          ((startPos >= r.startOffset && startPos < r.endOffset) ||
           (startPos + length > r.startOffset && startPos + length <= r.endOffset) ||
           (startPos <= r.startOffset && startPos + length >= r.endOffset))
        );
        return region?.type;
      };

      const drawSpecialLine = (text: string, x: number, y: number, lineIndex: number, layerId: number): boolean => {
        if (text.includes('[!]')) {
          const parts = text.split(/(\[!\])/);
          let currentX = x;
          let partOffset = 0;
          for (const part of parts) {
            const width = ctx.measureText(part).width;
            const censorType = getCensorType(lineIndex, partOffset, part.length, layerId);
            drawTextWithOutline(part, currentX, y, part === '[!]' ? 'rgb(255, 0, 195)' : 'rgb(255, 255, 255)', censorType);
            currentX += width;
            partOffset += part.length;
          }
          return true;
        }

        if (text.includes('[Character kill]')) {
          const parts = text.split(/(\[Character kill\])/);
          let currentX = x;
          let partOffset = 0;
          for (const part of parts) {
            const width = ctx.measureText(part).width;
            const censorType = getCensorType(lineIndex, partOffset, part.length, layerId);
            drawTextWithOutline(part, currentX, y, part === '[Character kill]' ? 'rgb(56, 150, 243)' : 'rgb(240, 0, 0)', censorType);
            currentX += width;
            partOffset += part.length;
          }
          return true;
        }

        return false;
      };

      const drawTextSegment = async (text: string, xPos: number, yPos: number, color: string, lineStartPos: number, lineIndex: number, layerId: number) => {
        let currentX = xPos;
        let remainingText = text;
        let currentOffset = lineStartPos;

        while (remainingText.length > 0) {
          const nextCensor = censoredRegions.value.find(region =>
            region.layerId === layerId &&
            region.lineIndex === lineIndex &&
            ((region.startOffset >= currentOffset && region.startOffset < currentOffset + remainingText.length) ||
             (region.endOffset > currentOffset && region.endOffset <= currentOffset + remainingText.length))
          );

          if (nextCensor) {
            const uncensoredLength = nextCensor.startOffset - currentOffset;
            if (uncensoredLength > 0) {
              const uncensoredText = remainingText.slice(0, uncensoredLength);
              drawTextWithOutline(uncensoredText, currentX, yPos, color);
              currentX += ctx.measureText(uncensoredText).width;
              currentOffset += uncensoredLength;
              remainingText = remainingText.slice(uncensoredLength);
            }

            const censoredLength = nextCensor.endOffset - currentOffset;
            if (censoredLength > 0) {
              const censoredText = remainingText.slice(0, censoredLength);
              if (nextCensor.type === CensorType.BlackBar) {
                const censorWidth = ctx.measureText(censoredText).width;
                ctx.fillStyle = '#000000';
                ctx.fillRect(currentX - 2, yPos, censorWidth + 4, 16);
              } else if (nextCensor.type === CensorType.Blur) {
                drawTextWithOutline(censoredText, currentX, yPos, color, nextCensor.type);
              } else if (nextCensor.type === CensorType.Invisible) {
                const spaceWidth = ctx.measureText(censoredText).width;
                currentX += spaceWidth;
              }

              currentOffset += censoredLength;
              remainingText = remainingText.slice(censoredLength);
              currentX += ctx.measureText(censoredText).width;
            }
          } else {
            if (remainingText.length > 0) {
              drawTextWithOutline(remainingText, currentX, yPos, color);
              currentX += ctx.measureText(remainingText).width;
              remainingText = '';
            }
          }
        }
      };

      let lineIndex = 0;
      for (const line of layer.parsedLines) {
        const rawText = stripHtml(line.text);
        const words = rawText.split(' ');
        let currentLineText = '';
        let lineStartPosition = 0;
        const maxTextWidth = chatLineWidth.value;

        const fullTextWidth = ctx.measureText(rawText).width;
        const textColor = line.color || getTextColor(rawText);
        if (fullTextWidth <= maxTextWidth) {
          drawBlackBar(currentY, fullTextWidth);
          if (!drawSpecialLine(rawText, 4, currentY, lineIndex, layer.id)) {
            await drawTextSegment(rawText, 4, currentY, textColor, lineStartPosition, lineIndex, layer.id);
          }
          currentY += 16;
          lineStartPosition += rawText.length;
          lineIndex++;
          continue;
        }

        for (const word of words) {
          const testLine = currentLineText ? currentLineText + ' ' + word : word;
          const metrics = ctx.measureText(testLine);

          if (metrics.width > maxTextWidth && currentLineText) {
            const currentLineWidth = ctx.measureText(currentLineText).width;
            drawBlackBar(currentY, currentLineWidth);
            if (!drawSpecialLine(currentLineText, 4, currentY, lineIndex, layer.id)) {
              await drawTextSegment(currentLineText, 4, currentY, textColor, lineStartPosition, lineIndex, layer.id);
            }
            currentY += 16;
            lineStartPosition += currentLineText.length + 1;
            currentLineText = word;
          } else {
            currentLineText = testLine;
          }
        }

        if (currentLineText) {
          const currentLineWidth = ctx.measureText(currentLineText).width;
          drawBlackBar(currentY, currentLineWidth);
          if (!drawSpecialLine(currentLineText, 4, currentY, lineIndex, layer.id)) {
            await drawTextSegment(currentLineText, 4, currentY, textColor, lineStartPosition, lineIndex, layer.id);
          }
          currentY += 16;
        }

        lineIndex++;
      }

      ctx.restore();
    }

    const filename = buildScreenshotFilename();
    const blob = await new Promise((resolve) => {
      canvas.toBlob(resolve, 'image/png');
    });

    if (!blob) {
      throw new Error('Failed to create blob');
    }

    await saveBlob(blob, filename);

  } catch (error) {
    console.error('Error saving image:', error);
  }
};

// Reset all state to default values
const resetSession = () => {
  // Reset image and transform
  droppedImageSrc.value = null;
  imageTransform.x = 0;
  imageTransform.y = 0;
  imageTransform.scale = 1;
  
  // Reset image dragging state
  isImageDraggingEnabled.value = false;
  isPanning.value = false;
  
  // Reset chat
  chatlogText.value = '';
  parsedChatLines.value = [];
  chatLayers.value = [];
  const defaultLayer = createChatLayer({ name: 'Layer 1' });
  selectedChatLayerId.value = defaultLayer.id;
  Object.assign(chatTransform, defaultLayer.transform);
  
  // Reset drop zone dimensions to defaults
  dropZoneWidth.value = 800;
  dropZoneHeight.value = 600;
  
  // Reset chat transform
  chatTransform.x = 0;
  chatTransform.y = 0;
  chatTransform.scale = 1;
  isChatDraggingEnabled.value = false;
  isChatPanning.value = false;

  // Reset censoring data
  censoredRegions.value = [];
  selectedText.lineIndex = -1;
  selectedText.layerId = -1;
  selectedText.startOffset = 0;
  selectedText.endOffset = 0;
  selectedText.text = '';
  renderKey.value++;
  
  // Close the confirmation dialog
  showNewSessionDialog.value = false;
};

// Add state for black bars toggle
const showBlackBars = ref(false);

// Toggle black bars
const toggleBlackBars = () => {
  showBlackBars.value = !showBlackBars.value;
};

// Censor types enum
enum CensorType {
  None = 'none',
  Invisible = 'invisible',
  BlackBar = 'blackbar',
  Blur = 'blur'
}

// Interface for censored regions
interface CensoredRegion {
  lineIndex: number;
  startOffset: number;
  endOffset: number;
  type: CensorType;
  layerId: number;
}

// State for censored regions
const censoredRegions = ref<CensoredRegion[]>([]);
const selectedText = reactive({
  layerId: -1,
  lineIndex: -1,
  startOffset: 0,
  endOffset: 0,
  text: ''
});

// Add a key to force re-render when censoring changes
const renderKey = ref(0);

// Update cycleCensorType to trigger re-render
const cycleCensorType = () => {
  if (selectedText.lineIndex === -1) return;
  if (selectedText.layerId === -1) return;

  console.log('Applying censoring:', selectedText);

  // Find existing censor for this region
  const existingIndex = censoredRegions.value.findIndex(
    region => region.layerId === selectedText.layerId &&
             region.lineIndex === selectedText.lineIndex &&
             region.startOffset === selectedText.startOffset &&
             region.endOffset === selectedText.endOffset
  );

  if (existingIndex === -1) {
    // Add new censor starting with Invisible
    console.log('Adding new censor region');
    censoredRegions.value.push({
      lineIndex: selectedText.lineIndex,
      startOffset: selectedText.startOffset,
      endOffset: selectedText.endOffset,
      type: CensorType.Invisible,
      layerId: selectedText.layerId
    });
  } else {
    const current = censoredRegions.value[existingIndex];
    console.log('Cycling existing censor:', current.type);
    switch (current.type) {
      case CensorType.Invisible:
        current.type = CensorType.BlackBar;
        break;
      case CensorType.BlackBar:
        current.type = CensorType.Blur;
        break;
      case CensorType.Blur:
        // Remove censoring
        censoredRegions.value.splice(existingIndex, 1);
        break;
    }
  }

  // Force re-render
  renderKey.value++;
  console.log('Updated censor regions:', censoredRegions.value);
};

// Update applyCensoring to handle partial text censoring
const applyCensoring = (text: string, lineIndex: number, layerId: number) => {
  const regions = censoredRegions.value
    .filter(region => region.layerId === layerId && region.lineIndex === lineIndex)
    .sort((a, b) => a.startOffset - b.startOffset);

  console.log(`Applying censoring to layer ${layerId}, line ${lineIndex}, found ${regions.length} regions`);

  if (regions.length === 0) return text;

  let result = '';
  let lastIndex = 0;

  for (const region of regions) {
    // Add uncensored text before this region
    result += text.slice(lastIndex, region.startOffset);
    
    // Add censored text
    const censoredText = text.slice(region.startOffset, region.endOffset);
    console.log('Censoring text:', censoredText, 'with type:', region.type);
    
    // Ensure the text is properly escaped for HTML
    const escapedText = censoredText
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#039;');
    
    switch (region.type) {
      case CensorType.Invisible:
        result += `<span class="censored-invisible">${escapedText}</span>`;
        break;
      case CensorType.BlackBar:
        result += `<span class="censored-blackbar">${escapedText}</span>`;
        break;
      case CensorType.Blur:
        result += `<span class="censored-blur">${escapedText}</span>`;
        break;
    }
    
    lastIndex = region.endOffset;
  }

  // Add remaining uncensored text
  result += text.slice(lastIndex);
  return result;
};

// Update handleTextSelection to work with textarea
const handleTextSelection = () => {
  const selection = window.getSelection();
  // Avoid noisy logs; this handler runs on every cursor move
  
  if (!selection || selection.toString().trim() === '') {
    selectedText.lineIndex = -1;
    selectedText.layerId = -1;
    return;
  }

  try {
    // Get the textarea element
    const textarea = document.querySelector('.chatlog-textarea textarea') as HTMLTextAreaElement;
    if (!textarea) return;

    const selectedValue = selection.toString().trim();
    const fullText = textarea.value;
    const lines = fullText.split('\n');
    
    // Find which line contains the selection
    let currentPos = 0;
    for (let i = 0; i < lines.length; i++) {
      const lineLength = lines[i].length + 1; // +1 for newline
      const lineStart = currentPos;
      const lineEnd = currentPos + lineLength;
      
      // Check if selection is in this line
      const selectionStart = textarea.selectionStart;
      if (selectionStart >= lineStart && selectionStart < lineEnd) {
        console.log('Found selection in line:', i, lines[i]);
        selectedText.lineIndex = i;
        selectedText.startOffset = selectionStart - lineStart;
        selectedText.endOffset = textarea.selectionEnd - lineStart;
        selectedText.text = selectedValue;
        selectedText.layerId = activeChatLayer.value?.id ?? -1;
        break;
      }
      
      currentPos += lineLength;
    }
    
    console.log('Selection state:', selectedText);
  } catch (error) {
    console.error('Error handling text selection:', error);
  }
};

// Helper function to strip HTML and extract text content
const stripHtml = (html: string): string => {
  const temp = document.createElement('div');
  temp.innerHTML = html;
  return temp.textContent || temp.innerText || '';
};

// Save editor state to cookie
const saveEditorState = () => {
  const state = {
    characterName: characterName.value,
    chatLayers: chatLayers.value.map(layer => ({
      id: layer.id,
      name: layer.name,
      text: layer.text,
      transform: { ...layer.transform },
      visible: layer.visible
    })),
    selectedChatLayerId: selectedChatLayerId.value,
    dropZoneWidth: dropZoneWidth.value,
    dropZoneHeight: dropZoneHeight.value,
    imageTransform: { ...imageTransform },
    isImageDraggingEnabled: isImageDraggingEnabled.value,
    isChatDraggingEnabled: isChatDraggingEnabled.value,
    showBlackBars: showBlackBars.value,
    censoredRegions: censoredRegions.value,
    selectedText: { ...selectedText },
    stripTimestamps: stripTimestamps.value,
    chatLineWidth: chatLineWidth.value,
    screenshotTheme: screenshotTheme.value,
    alwaysPromptSaveLocation: alwaysPromptSaveLocation.value
  };
  Cookies.set('editorState', JSON.stringify(state), { expires: 365 });
};

// Load editor state from cookie
const loadEditorState = () => {
  const savedState = Cookies.get('editorState');
  if (savedState) {
    try {
      const state = JSON.parse(savedState);
      characterName.value = state.characterName || '';
      dropZoneWidth.value = state.dropZoneWidth || 800;
      dropZoneHeight.value = state.dropZoneHeight || 600;
      Object.assign(imageTransform, state.imageTransform || { x: 0, y: 0, scale: 1 });
      isImageDraggingEnabled.value = state.isImageDraggingEnabled || false;
      isChatDraggingEnabled.value = state.isChatDraggingEnabled || false;
      showBlackBars.value = state.showBlackBars || false;
      const loadedRegions = state.censoredRegions || [];
      censoredRegions.value = loadedRegions.map((region: any) => ({
        layerId: region.layerId ?? activeChatLayer.value?.id ?? -1,
        lineIndex: region.lineIndex ?? -1,
        startOffset: region.startOffset ?? 0,
        endOffset: region.endOffset ?? 0,
        type: region.type ?? CensorType.Invisible
      }));
      Object.assign(selectedText, state.selectedText || { layerId: -1, lineIndex: -1, startOffset: 0, endOffset: 0, text: '' });
      selectedText.layerId = selectedText.layerId ?? -1;
      stripTimestamps.value = state.stripTimestamps || false; // Load the new option
      chatLineWidth.value = state.chatLineWidth || 640;
      screenshotTheme.value = state.screenshotTheme || '';
      alwaysPromptSaveLocation.value = state.alwaysPromptSaveLocation || false;

      if (state.chatLayers && Array.isArray(state.chatLayers)) {
        chatLayers.value = [];
        for (const layer of state.chatLayers) {
          const restored = createChatLayer({
            id: layer.id,
            name: layer.name,
            text: layer.text,
            transform: layer.transform,
            visible: layer.visible
          });
          parseChatlog(restored);
          nextChatLayerId = Math.max(nextChatLayerId, restored.id + 1);
        }
        const fallbackId = state.selectedChatLayerId ?? (chatLayers.value[0]?.id ?? null);
        if (fallbackId !== null) {
          selectChatLayer(fallbackId);
        }
      } else {
        // Legacy state: single chatlogText
        chatlogText.value = state.chatlogText || '';
        parseChatlog();
      }
      
    } catch (error) {
      console.error('Error loading editor state:', error);
    }
  }
};

// Load character name from cookie
const loadCharacterName = () => {
  const savedName = Cookies.get('characterName');
  if (savedName) {
    characterName.value = savedName;
  }
};

// Initialize flex layout values to create a stable layout
const initializeLayoutValues = () => {
  // Ensure we're using string percentages for flex basis values
  chatPanelFlexBasis.value = '25%';
  mainContentFlexBasis.value = '75%';
  
  // Ensure consistent size of both panels
  console.log('Layout initialized with main content: 75%, chat panel: 25%');
};

// Watch for changes that should trigger state save
watch([
  chatLayers,
  selectedChatLayerId,
  dropZoneWidth,
  dropZoneHeight,
  () => ({ ...imageTransform }),
  isImageDraggingEnabled,
  isChatDraggingEnabled,
  showBlackBars,
  censoredRegions,
  () => ({ ...selectedText }),
  stripTimestamps,
  chatLineWidth,
  screenshotTheme,
  alwaysPromptSaveLocation
], () => {
  saveEditorState();
}, { deep: true });

// Load saved state when component mounts
onMounted(() => {
  loadCharacterName();
  loadEditorState();
  
  // Make sure layout is initialized correctly
  initializeLayoutValues();
  
  // Calculate initial scale
  setTimeout(() => {
    calculateDropzoneScale();
    
    // Add window resize listener
    window.addEventListener('resize', () => {
      calculateDropzoneScale();
    });
  }, 100);
});

// Add this new method to handle the click on drop zone
const handleDropZoneClick = (event: Event) => {
  if (!droppedImageSrc.value) {
    triggerFileInput();
  }
};

</script>

<template>
  <div class="magician-wrapper">
    <v-toolbar density="compact" color="surface-variant">
      <div class="toolbar-button-group">
        <v-tooltip text="New Session" location="bottom">
          <template v-slot:activator="{ props }">
            <v-btn 
              v-bind="props" 
              icon="mdi-plus-box-outline"
              @click="showNewSessionDialog = true"
            ></v-btn>
          </template>
        </v-tooltip>
        <v-tooltip text="Import Chatlog" location="bottom">
          <template v-slot:activator="{ props }">
            <v-btn v-bind="props" icon="mdi-message-plus-outline" @click="triggerChatFileInput"></v-btn>
          </template>
        </v-tooltip>
        <v-tooltip text="Strip Timestamps from Chatlog" location="bottom">
          <template v-slot:activator="{ props }">
            <v-switch
              v-bind="props"
              v-model="stripTimestamps"
              color="primary"
              density="compact"
              hide-details
              class="ms-2 me-1 custom-switch"
              @change="parseChatlog" 
            ></v-switch>
          </template>
        </v-tooltip>
        <v-tooltip text="Import Layer Image (Coming Soon!)" location="bottom">
          <template v-slot:activator="{ props }">
            <v-btn v-bind="props" icon="mdi-layers-search" disabled></v-btn>
          </template>
        </v-tooltip>
      </div>

      <v-spacer></v-spacer>

      <div class="toolbar-button-group">
        <v-text-field
          v-model="characterName"
          label="Character Name"
          density="compact"
          hide-details
          variant="solo-filled"
          flat
          style="max-width: 300px; min-width: 250px;"
          class="mx-1"
          @input="parseChatlog"
          prepend-inner-icon="mdi-account"
        ></v-text-field>
      </div>

      <v-spacer></v-spacer>

      <div class="toolbar-button-group">
        <v-text-field
          v-model="screenshotTheme"
          label="Screenshot Name"
          density="compact"
          hide-details
          variant="solo-filled"
          flat
          style="max-width: 260px; min-width: 200px;"
          class="mx-1"
          prepend-inner-icon="mdi-tag-text-outline"
          placeholder="screenshot"
        ></v-text-field>
        <v-tooltip
          :text="filePickerSupported ? 'Ask where to save each time (uses browser Save dialog)' : 'Save dialog not supported in this browser'"
          location="bottom"
        >
          <template v-slot:activator="{ props }">
            <v-switch
              v-bind="props"
              v-model="alwaysPromptSaveLocation"
              :disabled="!filePickerSupported"
              color="primary"
              density="compact"
              hide-details
              class="ms-2 me-1 custom-switch"
              inset
              aria-label="Prompt for save location"
            ></v-switch>
          </template>
        </v-tooltip>
      </div>

      <v-spacer></v-spacer>

      <div class="toolbar-button-group">
        <v-text-field
          v-model.number="dropZoneWidth"
          label="Width"
          type="number"
          density="compact"
          hide-details
          variant="solo-filled"
          flat
          style="max-width: 120px;"
          prepend-inner-icon="mdi-arrow-expand-horizontal"
        ></v-text-field>
        <v-text-field
          v-model.number="dropZoneHeight"
          label="Height"
          type="number"
          density="compact"
          hide-details
          variant="solo-filled"
          flat
          style="max-width: 120px;"
          prepend-inner-icon="mdi-arrow-expand-vertical"
        ></v-text-field>
        <v-text-field
          v-model.number="chatLineWidth"
          label="Line Width"
          type="number"
          density="compact"
          hide-details
          variant="solo-filled"
          flat
          style="max-width: 120px;"
          prepend-inner-icon="mdi-format-line-spacing"
          min="300"
          max="1200"
          @change="renderKey++"
        ></v-text-field>
      </div>

      <v-spacer></v-spacer>

      <div class="toolbar-button-group">
        <v-tooltip text="Enable Image Drag/Zoom" location="bottom">
          <template v-slot:activator="{ props }">
            <v-btn 
              v-bind="props" 
              icon="mdi-drag-variant"
              :color="isImageDraggingEnabled ? 'primary' : undefined"
              @click="toggleImageDrag"
            ></v-btn>
          </template>
        </v-tooltip>
        <v-tooltip text="Enable Chat Drag/Zoom" location="bottom">
          <template v-slot:activator="{ props }">
            <v-btn 
              v-bind="props" 
              icon="mdi-message-text-lock-outline"
              :color="isChatDraggingEnabled ? 'primary' : undefined"
              @click="toggleChatDrag"
            ></v-btn>
          </template>
        </v-tooltip>
        <v-tooltip text="Enable Layer Drag (Coming Soon!)" location="bottom">
          <template v-slot:activator="{ props }">
            <v-btn v-bind="props" icon="mdi-layers-triple-outline" disabled></v-btn>
          </template>
        </v-tooltip>
      </div>

      <v-spacer></v-spacer>

      <div class="toolbar-button-group">
        <v-tooltip text="Toggle Black Bars" location="bottom">
          <template v-slot:activator="{ props }">
            <v-btn 
              v-bind="props" 
              icon="mdi-view-day-outline"
              :color="showBlackBars ? 'primary' : undefined"
              @click="toggleBlackBars"
            ></v-btn>
          </template>
        </v-tooltip>
        <v-tooltip text="Create Layer (Coming Soon!)" location="bottom">
          <template v-slot:activator="{ props }">
            <v-btn v-bind="props" icon="mdi-layers-plus" disabled></v-btn>
          </template>
        </v-tooltip>
        <v-tooltip text="Remove Layer (Coming Soon!)" location="bottom">
          <template v-slot:activator="{ props }">
            <v-btn v-bind="props" icon="mdi-layers-minus" disabled></v-btn>
          </template>
        </v-tooltip>
        <v-tooltip text="Censor Selection" location="bottom">
          <template v-slot:activator="{ props }">
            <v-btn 
              v-bind="props" 
              icon="mdi-eye-off"
              @click="cycleCensorType"
              :disabled="selectedText.lineIndex === -1"
            ></v-btn>
          </template>
        </v-tooltip>
        <v-tooltip text="Save Image" location="bottom">
          <template v-slot:activator="{ props }">
            <v-btn 
              v-bind="props" 
              icon="mdi-content-save-outline"
              @click="saveImage"
              :disabled="!droppedImageSrc"
            ></v-btn>
          </template>
        </v-tooltip>
      </div>
    </v-toolbar>

    <!-- Added Alert Banner for Known Issue -->
    <div class="alert-banner-container">
      <v-alert
        type="error"
        variant="tonal"
        prominent
        closable
        icon="mdi-alert"
        class="ma-0"
        border="start"
      >
        <strong>Known Issue: We are aware of inconsistencies affecting some exported images and are actively working on a fix. Thank you for your patience.</strong>
      </v-alert>
    </div>

    <!-- New Session Confirmation Dialog -->
    <v-dialog v-model="showNewSessionDialog" max-width="400">
      <v-card>
        <v-card-title class="text-h6">
          Start New Session?
        </v-card-title>
        <v-card-text>
          This will clear all current work, including any loaded images and chat data. This action cannot be undone.
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn
            color="grey-darken-1"
            variant="text"
            @click="showNewSessionDialog = false"
          >
            Cancel
          </v-btn>
          <v-btn
            color="red-darken-1"
            variant="text"
            @click="resetSession"
          >
            Reset Everything
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- Hidden file inputs -->
    <input 
      type="file" 
      ref="fileInputRef"
      class="hidden-input" 
      accept="image/*" 
      @change="handleFileSelect"
    />
    <input 
      type="file" 
      ref="chatFileInputRef"
      class="hidden-input" 
      accept=".txt,text/plain"
      @change="handleChatFileSelect"
    />

    <!-- Row takes remaining height and full width. Padding applied here. -->
    <div class="layout-container pa-2">
      <div class="main-content" ref="contentAreaRef" :style="{ width: mainContentFlexBasis }">
        <!-- Add aspect ratio container to enforce proper ratio -->
        <div class="aspect-ratio-container" :style="aspectRatioContainerStyle">
          <v-sheet 
            class="drop-zone d-flex align-center justify-center pa-0"
            :class="{ 
              'is-dragging-over': isDraggingOverDropZone,
              'clickable': !droppedImageSrc 
            }" 
            :style="dropZoneStyle as CSSProperties"
            @dragover.prevent.stop="handleDragOver"
            @dragleave="handleDragLeave"
            @dragenter.prevent.stop="handleDragOver"
            @drop.prevent.stop="handleDrop"
            @wheel="handleWheel"
            @click="handleDropZoneClick"
            :role="!droppedImageSrc ? 'button' : undefined"
            :tabindex="!droppedImageSrc ? 0 : undefined"
            @keydown.enter="handleDropZoneClick"
          >
            <!-- Scale indicator when dropzone is scaled down -->
            <div v-if="isScaledDown" class="scale-indicator">
              {{ scaleIndicator }}
            </div>
            
            <!-- Display Dropped Image -->
            <img 
              v-if="droppedImageSrc"
              :src="droppedImageSrc" 
              alt="Dropped Screenshot"
              class="dropped-image"
              :style="imageStyle as CSSProperties"
              draggable="false" 
              @mousedown="handleImageMouseDown"
              @mousemove="handleImageMouseMove"
              @mouseup="handleImageMouseUpOrLeave"
              @mouseleave="handleImageMouseUpOrLeave"
            />
            <!-- Display Placeholder -->
            <div v-else class="text-center">
              <v-icon size="x-large" color="grey-darken-1">mdi-paperclip</v-icon>
              <div class="text-grey-darken-1 mt-2">Click or drag and drop your screenshot here</div>
            </div>

            <!-- Chat Overlay with fixed-width wrapping -->
            <template v-for="layer in chatLayers">
              <div
                v-if="droppedImageSrc && layer && layer.visible && layer.parsedLines && layer.parsedLines.length > 0"
                :key="`chat-layer-${layer.id}-${renderKey}`"
                class="chat-overlay"
                @mousedown="handleChatMouseDown"
                @mousemove="handleChatMouseMove"
                @mouseup="handleChatMouseUpOrLeave"
                @mouseleave="handleChatMouseUpOrLeave"
                @wheel="handleChatWheel"
                :style="chatStyles(layer)"
              >
                <div class="chat-lines-container">
                  <div
                    v-for="(line, index) in layer.parsedLines"
                    :key="line.id"
                    class="chat-line"
                    :style="{ color: line.color }"
                  >
                    <span v-if="showBlackBars" class="with-black-bar">
                      <span class="chat-text" v-html="applyCensoring(line.text, index, layer.id)" @mouseup="handleTextSelection"></span>
                    </span>
                    <span v-else class="chat-text" v-html="applyCensoring(line.text, index, layer.id)" @mouseup="handleTextSelection"></span>
                  </div>
                </div>
              </div>
            </template>
          </v-sheet>
        </div>
      </div>

      <!-- Add resize handle between columns -->
      <div 
        class="resize-handle"
        @mousedown.prevent="handleResizeMouseDown"
      ></div>

      <!-- Right side chatlog panel -->
      <div class="chatlog-panel" ref="chatPanelRef" :style="{ width: chatPanelFlexBasis }">
        <v-sheet class="fill-height d-flex flex-column pa-2" style="border-radius: 4px;">
          <div class="d-flex align-center mb-1">
            <div class="text-subtitle-1">Chat Layers</div>
            <v-spacer></v-spacer>
            <v-btn icon="mdi-plus" size="small" variant="text" @click="addChatLayer"></v-btn>
          </div>
          <div class="mb-2" style="max-height: 180px; overflow-y: auto;">
            <v-list density="compact" bg-color="transparent">
              <v-list-item
                v-for="(layer, index) in chatLayers"
                :key="layer.id"
                :active="layer.id === activeChatLayer?.id"
                @click="selectChatLayer(layer.id)"
              >
                <template #prepend>
                  <v-btn
                    icon
                    size="small"
                    variant="text"
                    :color="layer.visible ? 'primary' : undefined"
                    :icon="layer.visible ? 'mdi-eye-outline' : 'mdi-eye-off-outline'"
                    @click.stop="layer.visible = !layer.visible"
                  ></v-btn>
                </template>
                <v-text-field
                  v-model="layer.name"
                  variant="underlined"
                  density="compact"
                  hide-details
                  class="mx-2"
                  placeholder="Layer name"
                ></v-text-field>
                <template #append>
                  <v-btn icon size="small" variant="text" :disabled="index === 0" @click.stop="moveChatLayer(layer.id, 'up')">
                    <v-icon size="small">mdi-arrow-up</v-icon>
                  </v-btn>
                  <v-btn icon size="small" variant="text" :disabled="index === chatLayers.length - 1" @click.stop="moveChatLayer(layer.id, 'down')">
                    <v-icon size="small">mdi-arrow-down</v-icon>
                  </v-btn>
                  <v-btn icon size="small" variant="text" :disabled="chatLayers.length === 1" @click.stop="removeChatLayer(layer.id)">
                    <v-icon size="small">mdi-delete-outline</v-icon>
                  </v-btn>
                </template>
              </v-list-item>
            </v-list>
          </div>

          <div class="text-subtitle-1 mb-1">Chatlog Snippet</div>
          <div class="flex-grow-1 d-flex flex-column" style="overflow-y: hidden;">
            <v-textarea
              v-model="chatlogText"
              placeholder="Paste your chat lines here!&#10;Hit 'Parse' afterwards to see color-swatched parsed lines on your image."
              class="chatlog-textarea mb-1"
              density="compact"
              variant="outlined"
              hide-details
              no-resize
              @keyup.ctrl.enter="parseChatlog"
              @mouseup="handleTextSelection"
              @select="handleTextSelection"
              @keyup="handleTextSelection"
            ></v-textarea>
          </div>
          <div class="mt-auto pt-1">
            <v-row no-gutters>
              <v-col class="pe-1">
                <v-btn
                  color="grey-darken-3"
                  block
                  @click="clearChatlog"
                  density="compact"
                >
                  Clear
                </v-btn>
              </v-col>
              <v-col class="ps-1">
                <v-btn
                  color="grey-darken-3"
                  block
                  @click="parseChatlog"
                  density="compact"
                >
                  Parse
                </v-btn>
              </v-col>
            </v-row>
          </div>
        </v-sheet>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Add explicit fixed-width style for chat display */
.chat-lines-container {
  position: relative;
  width: 100%; 
  padding: 0;
  margin: 0;
}

.chat-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: auto;
  pointer-events: auto;
  overflow: visible;
  transform-origin: top left;
  will-change: transform;
}

.chat-line {
  position: relative;
  display: block;
  font-family: Arial, sans-serif;
  font-size: 12px;
  line-height: 1.3;
  padding: 0;
  white-space: pre-wrap;
  overflow-wrap: break-word;
  width: v-bind('chatLineWidth + "px"');
  margin: 0;
  text-shadow: 
    -1px -1px 0 #000,
    -1px 0 0 #000,
    -1px 1px 0 #000,
    0 -1px 0 #000,
    0 1px 0 #000,
    1px -1px 0 #000,
    1px 0 0 #000,
    1px 1px 0 #000;
  -webkit-font-smoothing: none !important;
  font-weight: 700;
  letter-spacing: 0;
}

.chat-text {
  display: inline;
  padding-right: 5px;
  user-select: text !important;
  cursor: text;
  -webkit-box-decoration-break: clone;
  box-decoration-break: clone;
}

.with-black-bar {
  background-color: #000000;
  padding: 0 4px;
  display: inline; /* Changed from block to inline */
  box-decoration-break: clone;
  -webkit-box-decoration-break: clone;
}

/* Keep other CSS styles the same */
.magician-wrapper {
  background-color: rgb(var(--v-theme-surface-variant));
  height: 100%;
  width: 100%;
  display: flex;
  flex-direction: column;
}

.drop-zone {
  border: 2px dashed transparent;
  transition: background-color 0.2s ease, border-color 0.2s ease;
}

.drop-zone.clickable:hover {
  border-color: #42a5f5;
  background-color: #e3f2fd;
  cursor: pointer;
}

.drop-zone.clickable:focus {
  outline: none;
  border-color: #42a5f5;
  background-color: #e3f2fd;
}

.dropped-image {
  display: block;
  user-select: none; /* Prevent image selection during drag */
  -webkit-user-drag: none; /* Prevent browser native image drag */
  position: absolute; /* Ensure it fits within the container */
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: contain; /* Scale image while preserving aspect ratio */
}

.chatlog-textarea {
  height: 100%;
  user-select: text !important;
}

:deep(.chatlog-textarea textarea) {
  user-select: text !important;
  cursor: text;
  height: 100% !important;
  max-height: none !important;
}

.hidden-input {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  border: 0;
}

/* Censoring styles */
:deep(.censored-invisible) {
  opacity: 0;
  user-select: text;
  display: inline-block;
  background-color: transparent;
}

:deep(.censored-blackbar) {
  background-color: #000000;
  color: transparent;
  user-select: text;
  display: inline-block;
  padding: 0 2px;
}

:deep(.censored-blur) {
  filter: blur(5px);
  user-select: text;
  display: inline-block;
  padding: 0 2px;
}

.fill-height {
  height: 100%;
}

.toolbar-button-group .v-btn {
  margin: 0 2px;
}

.toolbar-button-group .v-text-field {
  margin: 0 4px;
}

.toolbar-button-group {
  display: flex;
  align-items: center; /* Align text fields vertically */
}

.resize-handle {
  width: 8px; /* Even wider for easier targeting */
  cursor: col-resize;
  background-color: rgba(128, 128, 128, 0.5);
  height: 100%;
  flex-shrink: 0;
  transition: background-color 0.2s ease;
  position: relative; /* Added for pseudo-element positioning */
  z-index: 1; /* Ensure it's above other elements */
}

.resize-handle:hover {
  background-color: rgba(128, 128, 128, 0.8);
}

/* Add visual indicator in the middle of the handle */
.resize-handle::after {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  height: 80px; /* Taller indicator */
  width: 2px;
  background-color: rgba(200, 200, 200, 0.9); /* Brighter color */
  border-radius: 1px;
}

/* Add dots to make it more visible as a grip */
.resize-handle::before {
  content: ':::';
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%) rotate(90deg);
  font-size: 12px;
  color: rgba(240, 240, 240, 0.7); /* Light colored dots */
  letter-spacing: 2px;
}

.layout-container {
  display: flex;
  flex-direction: row;
  width: 100%;
  height: 100%;
  overflow: hidden;
  position: relative; /* Ensure proper stacking context */
}

.main-content {
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 4px;
}

.chatlog-panel {
  height: 100%;
  padding: 4px;
}

.aspect-ratio-container {
  position: relative;
  width: 100%;
  height: 0; /* Height will be determined by padding-top */
  max-height: 100%;
  padding-bottom: 0;
  overflow: visible;
}

/* Add styles for the drop-zone when inside an aspect ratio container */
.aspect-ratio-container .drop-zone {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%) scale(var(--container-scale, 1));
  transform-origin: center center;
}

.scale-indicator {
  position: absolute;
  top: 10px;
  right: 10px;
  background-color: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 12px;
  z-index: 10;
  pointer-events: none;
  border: 1px solid rgba(255, 255, 255, 0.3);
}

/* Add custom style for the switch if needed */
.custom-switch {
  /* Adjust display or margins if it doesn't align well in the toolbar */
  display: inline-flex; /* Helps with alignment in flex containers */
  align-items: center;
}
</style>

