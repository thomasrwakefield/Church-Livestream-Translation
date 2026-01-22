# Live Church Service Captioning & Translation Feature

## Overview

This feature allows churches to caption and translate their services in real-time and display the translations in the mobile app. The system intercepts existing church streams and utilizes AI to handle captioning and translation.

---

## Architecture

### High-Level Flow

```
Church Stream (RTMP/HLS/YouTube) 
    ↓
Intercept & Extract Audio
    ↓
AI Transcription (Real-time)
    ↓
AI Translation (Multiple Languages)
    ↓
WebSocket/Server-Sent Events
    ↓
Mobile App Display (Live Captions)
```

---

## Implementation Approach: OpenAI Whisper + GPT-4

### Why This Approach?

- **Leverages existing OpenAI integration** in the codebase
- **Highly accurate transcription** (Whisper is state-of-the-art)
- **Multi-language translation** support (50+ languages via GPT-4)
- **No additional ML infrastructure** needed
- **Works with any stream format** (RTMP, HLS, YouTube)

### Technical Stack

- **Audio Extraction**: `ffmpeg` (Node.js)
- **Transcription**: OpenAI Whisper API
- **Translation**: OpenAI GPT-4 API
- **Real-time Delivery**: Socket.io (WebSockets)
- **Storage**: MongoDB
- **Mobile Display**: React Native

---

## Backend Implementation

### 1. New Service: `liveCaptionService.js`

**Location**: `webapp/server/services/liveCaptionService.js`

```javascript
const ffmpeg = require('fluent-ffmpeg');
const OpenAI = require('openai');
const openai = new OpenAI({ apiKey: '****hidden****' });
const fs = require('fs');

class LiveCaptionService {
  constructor() {
    this.activeStreams = new Map(); // Track active caption sessions
  }

  // Start capturing audio from stream
  async startCaptioning(streamUrl, organizationId, targetLanguages = ['en']) {
    const sessionId = generateHexString();
    
    // Extract audio in 10-second chunks
    const audioChunks = this.extractAudioChunks(streamUrl);
    
    // Process each chunk
    for await (const audioChunk of audioChunks) {
      // 1. Transcribe with Whisper
      const transcription = await this.transcribe(audioChunk);
      
      // 2. Translate to target languages
      const translations = await this.translateCaptions(
        transcription.text, 
        targetLanguages
      );
      
      // 3. Broadcast to connected clients
      this.broadcastCaptions(organizationId, {
        timestamp: Date.now(),
        original: transcription.text,
        translations: translations
      });
      
      // 4. Store for replay/archive
      await this.saveCaptions(organizationId, transcription, translations);
    }
  }

  // Extract audio using ffmpeg
  extractAudioChunks(streamUrl) {
    return new ReadableStream({
      start(controller) {
        ffmpeg(streamUrl)
          .audioCodec('libmp3lame')
          .audioChannels(1)
          .audioFrequency(16000) // Whisper optimized
          .format('mp3')
          .outputOptions('-segment_time 10') // 10-second chunks
          .on('data', (chunk) => {
            controller.enqueue(chunk);
          })
          .on('end', () => {
            controller.close();
          })
          .run();
      }
    });
  }

  // Transcribe using OpenAI Whisper
  async transcribe(audioBuffer) {
    const tempFile = `/tmp/audio-${Date.now()}.mp3`;
    fs.writeFileSync(tempFile, audioBuffer);
    
    const transcription = await openai.audio.transcriptions.create({
      file: fs.createReadStream(tempFile),
      model: 'whisper-1',
      language: 'en', // Auto-detect or specify
      response_format: 'verbose_json', // Includes timestamps
    });
    
    fs.unlinkSync(tempFile); // Cleanup
    
    return transcription;
  }

  // Translate captions using GPT-4
  async translateCaptions(text, targetLanguages) {
    const translations = {};
    
    for (const lang of targetLanguages) {
      const prompt = `Translate this church service caption to ${lang}. Keep it concise and natural:\n\n"${text}"`;
      
      const response = await openai.chat.completions.create({
        model: 'gpt-4',
        messages: [{ role: 'user', content: prompt }],
        max_tokens: 150,
        temperature: 0.3
      });
      
      translations[lang] = response.choices[0].message.content.trim();
    }
    
    return translations;
  }

  // Broadcast via WebSocket
  broadcastCaptions(organizationId, captionData) {
    // Use Socket.io to emit to all connected mobile clients
    io.to(`captions-${organizationId}`).emit('new-caption', captionData);
  }

  // Save to database for replay
  async saveCaptions(organizationId, transcription, translations) {
    await ServiceCaptions.insertOne({
      organization_id: ObjectId(organizationId),
      timestamp: new Date(),
      original_text: transcription.text,
      translations: translations,
      word_timestamps: transcription.words, // For fine-grained sync
    });
  }
}

module.exports = new LiveCaptionService();
```

---

### 2. API Endpoints

**New Controller**: `webapp/server/controllers/v2/services/captions.js`

```javascript
// Start live captioning for a service
module.exports.startLiveCaptions = async (parameters, user, req) => {
  const { stream_url, target_languages } = parameters;
  const organization_id = user.organisation_id;
  
  // Validate stream URL
  if (!stream_url) {
    throw { status: false, message: 'Stream URL required' };
  }
  
  // Start captioning service
  const session = await liveCaptionService.startCaptioning(
    stream_url,
    organization_id,
    target_languages || ['en', 'es', 'fr'] // Default languages
  );
  
  return {
    status: true,
    message: 'Live captioning started',
    session_id: session.id,
    available_languages: target_languages
  };
};

// Stop captioning
module.exports.stopLiveCaptions = async (parameters, user, req) => {
  const { session_id } = parameters;
  
  await liveCaptionService.stopCaptioning(session_id);
  
  return { status: true, message: 'Captioning stopped' };
};

// Get caption history (for replay)
module.exports.getCaptionHistory = async (parameters, user, req) => {
  const { organization_id, start_time, end_time } = parameters;
  
  const captions = await ServiceCaptions.find({
    organization_id: ObjectId(organization_id),
    timestamp: { $gte: new Date(start_time), $lte: new Date(end_time) }
  }).sort({ timestamp: 1 }).toArray();
  
  return { status: true, data: captions };
};
```

**Routes**: `webapp/server/routes/v2/services.js`

```javascript
router.post('/captions/start', controller.startLiveCaptions);
router.post('/captions/stop', controller.stopLiveCaptions);
router.get('/captions/history', controller.getCaptionHistory);
```

---

### 3. Database Schema

#### Collection: `service_captions`

```javascript
{
  _id: ObjectId,
  organization_id: ObjectId,
  session_id: String,
  timestamp: Date,
  original_text: String,
  original_language: String, // 'en'
  translations: {
    es: String,  // Spanish
    fr: String,  // French
    zh: String,  // Chinese
    ko: String,  // Korean
    pt: String,  // Portuguese
    // ... more languages as needed
  },
  word_timestamps: [
    { word: 'Hello', start: 0.5, end: 0.8 },
    { word: 'everyone', start: 0.9, end: 1.2 }
  ]
}
```

#### Collection: `caption_sessions`

```javascript
{
  _id: ObjectId,
  organization_id: ObjectId,
  stream_url: String,
  status: String, // 'active', 'stopped', 'error'
  target_languages: [String],
  started_at: Date,
  ended_at: Date,
  total_captions: Number,
  error_log: [String]
}
```

---

## Mobile App Implementation

### 1. WebSocket Hook

**Location**: `mobile-app/src/hooks/useLiveCaptions.ts`

```typescript
import { useEffect, useState } from 'react';
import io from 'socket.io-client';

interface Caption {
  text: string;
  timestamp: number;
}

interface CaptionData {
  timestamp: number;
  original: string;
  translations: { [key: string]: string };
}

export const useLiveCaptions = (organizationId: string, language: string) => {
  const [captions, setCaptions] = useState<Caption[]>([]);
  const [currentCaption, setCurrentCaption] = useState<string>('');
  const [socket, setSocket] = useState<any>(null);

  useEffect(() => {
    // Connect to WebSocket
    const newSocket = io('****hidden****', {
      auth: { token: '****hidden****' }
    });

    // Join organization's caption room
    newSocket.emit('join-captions', { organizationId });

    // Listen for new captions
    newSocket.on('new-caption', (data: CaptionData) => {
      const translatedText = data.translations[language] || data.original;
      
      setCurrentCaption(translatedText);
      setCaptions(prev => [...prev, {
        text: translatedText,
        timestamp: data.timestamp
      }]);

      // Auto-clear after 5 seconds
      setTimeout(() => setCurrentCaption(''), 5000);
    });

    setSocket(newSocket);

    return () => {
      newSocket.disconnect();
    };
  }, [organizationId, language]);

  return { currentCaption, captions, isConnected: socket?.connected };
};
```

---

### 2. Caption Display Component

**Location**: `mobile-app/src/screens/service/components/live-captions.tsx`

```tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';
import { useLiveCaptions } from '../../../hooks/useLiveCaptions';

interface LiveCaptionsProps {
  organizationId: string;
  selectedLanguage: string;
}

export const LiveCaptions: React.FC<LiveCaptionsProps> = ({ 
  organizationId, 
  selectedLanguage 
}) => {
  const { currentCaption, isConnected } = useLiveCaptions(
    organizationId, 
    selectedLanguage
  );

  if (!isConnected) {
    return <Text style={styles.status}>Connecting to captions...</Text>;
  }

  return (
    <View style={styles.captionContainer}>
      <Text style={styles.captionText}>{currentCaption}</Text>
    </View>
  );
};

const styles = StyleSheet.create({
  captionContainer: {
    position: 'absolute',
    bottom: 80,
    left: 20,
    right: 20,
    backgroundColor: 'rgba(0, 0, 0, 0.7)',
    padding: 15,
    borderRadius: 8,
    minHeight: 60,
  },
  captionText: {
    color: '#fff',
    fontSize: 18,
    textAlign: 'center',
    fontWeight: '600',
  },
  status: {
    color: '#666',
    fontSize: 14,
    textAlign: 'center',
  }
});
```

---

### 3. Language Selector Component

**Location**: `mobile-app/src/screens/service/components/language-selector.tsx`

```tsx
import React from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';

interface Language {
  code: string;
  name: string;
}

interface LanguageSelectorProps {
  onSelect: (code: string) => void;
  selectedLanguage: string;
}

export const LanguageSelector: React.FC<LanguageSelectorProps> = ({ 
  onSelect, 
  selectedLanguage 
}) => {
  const languages: Language[] = [
    { code: 'en', name: 'English' },
    { code: 'es', name: 'Español' },
    { code: 'fr', name: 'Français' },
    { code: 'zh', name: '中文' },
    { code: 'ko', name: '한국어' },
    { code: 'pt', name: 'Português' },
    { code: 'de', name: 'Deutsch' },
    { code: 'it', name: 'Italiano' },
    { code: 'ja', name: '日本語' },
    { code: 'ru', name: 'Русский' },
  ];

  return (
    <View style={styles.languageSelector}>
      {languages.map(lang => (
        <TouchableOpacity 
          key={lang.code}
          style={[
            styles.languageButton,
            selectedLanguage === lang.code && styles.selectedButton
          ]}
          onPress={() => onSelect(lang.code)}
        >
          <Text style={[
            styles.languageText,
            selectedLanguage === lang.code && styles.selectedText
          ]}>
            {lang.name}
          </Text>
        </TouchableOpacity>
      ))}
    </View>
  );
};

const styles = StyleSheet.create({
  languageSelector: {
    flexDirection: 'row',
    flexWrap: 'wrap',
    padding: 10,
    backgroundColor: '#f5f5f5',
  },
  languageButton: {
    padding: 10,
    margin: 5,
    borderRadius: 5,
    backgroundColor: '#fff',
    borderWidth: 1,
    borderColor: '#ddd',
  },
  selectedButton: {
    backgroundColor: '#007AFF',
    borderColor: '#007AFF',
  },
  languageText: {
    fontSize: 14,
    color: '#333',
  },
  selectedText: {
    color: '#fff',
    fontWeight: 'bold',
  }
});
```

---

## Angular Admin Panel

### Service Captions Component

**Location**: `angular-webapp/src/app/layouts/admin/service-captions/`

#### Component TypeScript

```typescript
import { Component, OnInit } from '@angular/core';
import { CaptionService } from '../../../services/caption.service';
import { ToastrService } from 'ngx-toastr';

@Component({
  selector: 'app-service-captions',
  templateUrl: './service-captions.component.html',
  styleUrls: ['./service-captions.component.css']
})
export class ServiceCaptionsComponent implements OnInit {
  streamUrl: string = '';
  isLive: boolean = false;
  sessionId: string = '';
  selectedLanguages: string[] = ['en', 'es'];
  
  availableLanguages = [
    { code: 'en', name: 'English' },
    { code: 'es', name: 'Spanish' },
    { code: 'fr', name: 'French' },
    { code: 'de', name: 'German' },
    { code: 'zh', name: 'Chinese' },
    { code: 'ko', name: 'Korean' },
    { code: 'pt', name: 'Portuguese' },
    { code: 'it', name: 'Italian' },
    { code: 'ja', name: 'Japanese' },
    { code: 'ru', name: 'Russian' },
  ];

  constructor(
    private captionService: CaptionService,
    private toastr: ToastrService
  ) {}

  ngOnInit() {
    this.loadStreamSettings();
  }

  async loadStreamSettings() {
    // Load saved stream URL and language preferences
    const settings = await this.captionService.getSettings();
    if (settings) {
      this.streamUrl = settings.stream_url || '';
      this.selectedLanguages = settings.target_languages || ['en', 'es'];
    }
  }

  toggleLanguage(languageCode: string) {
    const index = this.selectedLanguages.indexOf(languageCode);
    if (index > -1) {
      this.selectedLanguages.splice(index, 1);
    } else {
      this.selectedLanguages.push(languageCode);
    }
  }

  async startCaptions() {
    if (!this.streamUrl) {
      this.toastr.error('Please enter a stream URL');
      return;
    }

    if (this.selectedLanguages.length === 0) {
      this.toastr.error('Please select at least one language');
      return;
    }

    try {
      const response = await this.captionService.startLiveCaptions({
        stream_url: this.streamUrl,
        target_languages: this.selectedLanguages
      });
      
      this.isLive = true;
      this.sessionId = response.session_id;
      this.toastr.success('Live captions started successfully');
    } catch (error) {
      this.toastr.error('Failed to start captions: ' + error.message);
    }
  }

  async stopCaptions() {
    try {
      await this.captionService.stopLiveCaptions(this.sessionId);
      this.isLive = false;
      this.sessionId = '';
      this.toastr.success('Live captions stopped');
    } catch (error) {
      this.toastr.error('Failed to stop captions: ' + error.message);
    }
  }

  async testStream() {
    // Test if stream URL is valid
    try {
      const isValid = await this.captionService.testStreamUrl(this.streamUrl);
      if (isValid) {
        this.toastr.success('Stream URL is valid');
      } else {
        this.toastr.error('Invalid stream URL');
      }
    } catch (error) {
      this.toastr.error('Error testing stream: ' + error.message);
    }
  }
}
```

#### Component HTML

```html
<div class="service-captions-container">
  <h2>Live Service Captions</h2>
  
  <div class="stream-settings">
    <div class="form-group">
      <label for="streamUrl">Stream URL</label>
      <input 
        type="text" 
        id="streamUrl"
        [(ngModel)]="streamUrl" 
        placeholder="https://stream.example.com/live"
        [disabled]="isLive"
        class="form-control"
      />
      <button 
        *ngIf="!isLive"
        (click)="testStream()"
        class="btn btn-secondary btn-sm"
      >
        Test Stream
      </button>
    </div>

    <div class="form-group">
      <label>Target Languages</label>
      <div class="language-grid">
        <div 
          *ngFor="let lang of availableLanguages"
          class="language-option"
          [class.selected]="selectedLanguages.includes(lang.code)"
          (click)="!isLive && toggleLanguage(lang.code)"
        >
          <input 
            type="checkbox"
            [checked]="selectedLanguages.includes(lang.code)"
            [disabled]="isLive"
          />
          <span>{{ lang.name }}</span>
        </div>
      </div>
    </div>

    <div class="control-buttons">
      <button 
        *ngIf="!isLive"
        (click)="startCaptions()"
        class="btn btn-primary"
      >
        Start Live Captions
      </button>
      
      <button 
        *ngIf="isLive"
        (click)="stopCaptions()"
        class="btn btn-danger"
      >
        Stop Captions
      </button>
    </div>

    <div *ngIf="isLive" class="status-indicator">
      <span class="live-badge">🔴 LIVE</span>
      <p>Captions are broadcasting to mobile apps</p>
    </div>
  </div>
</div>
```

---

## Cost Analysis

### OpenAI API Pricing

| Service | Cost | Calculation |
|---------|------|-------------|
| **Whisper API** | $0.006/minute | ~$0.72 per 2-hour service |
| **GPT-4 Translation** | ~$0.03 per request | ~$0.30 per service (10 chunks) |
| **Total per Service** | | **~$1.00** |
| **Monthly (4 services)** | | **~$4-5/church** |

### Scaling Costs

- **100 churches**: ~$400-500/month
- **1,000 churches**: ~$4,000-5,000/month

**Note**: Costs decrease with volume and can be offset by per-church subscription pricing.

---

## Alternative Approach: AWS Transcribe + Translate

### Pros
- Lower latency (~1-2 seconds)
- Cheaper at scale ($0.0004/second = $1.44/hour)
- Purpose-built for streaming

### Cons
- Requires AWS account setup
- More complex integration
- Less accurate than Whisper for some accents

### Cost Comparison
- **AWS**: ~$1.44 per 2-hour service
- **OpenAI**: ~$1.00 per 2-hour service

---

## Implementation Phases

### Phase 1: Core Captioning (Week 1-2)
- [ ] Set up ffmpeg audio extraction
- [ ] Integrate OpenAI Whisper API
- [ ] Create WebSocket server for real-time delivery
- [ ] Build basic mobile display component
- [ ] Test with sample stream

### Phase 2: Translation (Week 2-3)
- [ ] Add GPT-4 translation pipeline
- [ ] Implement language selector UI
- [ ] Store caption history in MongoDB
- [ ] Add replay functionality

### Phase 3: Admin Controls (Week 3-4)
- [ ] Create Angular admin panel for stream setup
- [ ] Add start/stop controls
- [ ] Implement language configuration
- [ ] Build caption archive viewer

### Phase 4: Optimization (Week 4+)
- [ ] Reduce latency to < 3 seconds
- [ ] Implement auto-reconnect on disconnection
- [ ] Add caption styling options
- [ ] Enable offline caption download
- [ ] Add usage analytics dashboard

---

## Dependencies

### Backend (Node.js)
```json
{
  "socket.io": "^4.5.0",
  "fluent-ffmpeg": "^2.1.2",
  "openai": "^6.16.0" // Already installed
}
```

### Mobile App (React Native)
```json
{
  "socket.io-client": "^4.5.0"
}
```

### System Requirements
- **ffmpeg** installed on server
- **OpenAI API key** (already configured)

---

## Testing Strategy

### Unit Tests
- Audio extraction from various stream formats
- Whisper API transcription accuracy
- Translation quality across languages
- WebSocket connection handling

### Integration Tests
- End-to-end caption flow
- Multi-language broadcast
- Session management
- Error recovery

### Load Tests
- Multiple concurrent streams
- 100+ connected mobile clients
- Network interruption handling

---

## Security Considerations

1. **Stream URL Validation**: Verify church owns the stream
2. **WebSocket Authentication**: Require JWT tokens
3. **Rate Limiting**: Prevent API abuse
4. **Data Privacy**: Store captions with encryption
5. **Access Control**: Only organization members can view captions

---

## Future Enhancements

- [ ] Support for multiple simultaneous services
- [ ] Real-time editing of captions by admins
- [ ] Caption search and indexing
- [ ] Sermon transcript generation
- [ ] Integration with church website
- [ ] Downloadable caption files (SRT format)
- [ ] AI-powered sermon summaries
- [ ] Caption style customization (font, size, position)

---

## Support Documentation

### For Churches

**Stream URL Formats Supported:**
- RTMP: `rtmp://server/live/stream`
- HLS: `https://server/playlist.m3u8`
- YouTube Live: `https://youtube.com/watch?v=...`

**Setup Instructions:**
1. Navigate to Admin Panel > Service Captions
2. Enter your stream URL
3. Select target languages
4. Click "Start Live Captions"
5. Members can view captions in mobile app

### For Mobile Users

**How to View Captions:**
1. Open the service in the app
2. Tap the language selector icon
3. Choose your preferred language
4. Captions will appear at the bottom of the screen

---

## Technical Support

For issues or questions, contact:
- Backend: Check `server/services/liveCaptionService.js`
- Mobile: Check `src/hooks/useLiveCaptions.ts`
- Admin Panel: Check `src/app/layouts/admin/service-captions/`

---

**Last Updated**: January 22, 2026
