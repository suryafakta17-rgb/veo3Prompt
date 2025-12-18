# veo3Prompt
GeminiPro
import React, { useState, useEffect, useCallback } from 'react';
import { 
  Video, 
  Image as ImageIcon, 
  Users, 
  MapPin, 
  Clapperboard, 
  Move, 
  Camera, 
  Sun, 
  Music, 
  MessageSquare, 
  Ban, 
  Copy, 
  RefreshCw,
  Trash2,
  ChevronRight,
  ChevronLeft,
  Sparkles,
  Loader2
} from 'lucide-react';

const apiKey = ""; // Diset otomatis oleh environment

const App = () => {
  const [activeStep, setActiveStep] = useState(0);
  const [isGenerating, setIsGenerating] = useState(false);
  const [outputJson, setOutputJson] = useState(null);
  const [images, setImages] = useState([]);
  const [statusMessage, setStatusMessage] = useState("");

  const [formData, setFormData] = useState({
    metadata: {
      model: 'veo-3',
      duration: '7-8 seconds',
      aspect_ratio: '16:9',
      visual_style: 'cinematic',
      quality: 'high'
    },
    character_count: 1,
    characters: [
      { name: '', gender: '', age: '', hair: '', eyes: '', clothing: '', expression: '', emotion: '' },
      { name: '', gender: '', age: '', hair: '', eyes: '', clothing: '', expression: '', emotion: '' }
    ],
    environment: {
      location: '',
      time: '',
      weather: '',
      atmosphere: ''
    },
    scene: {
      description: '',
      action_sequence: '',
      interaction: ''
    },
    movement: {
      character_1: '',
      character_2: ''
    },
    camera: {
      shot_type: '',
      movement: '',
      start_frame: '',
      end_frame: '',
      stability: 'stable'
    },
    lighting: {
      type: '',
      direction: '',
      tone: ''
    },
    audio: {
      ambient: '',
      music: '',
      volume: 'balanced'
    },
    dialog: {
      enabled: false,
      language: 'English',
      timestamp: '00:01',
      text: '',
      voice_tone: '',
      emotion: ''
    }
  });

  const steps = [
    { id: 'metadata', label: 'Metadata', icon: <Video className="w-5 h-5" /> },
    { id: 'images', label: 'Reference', icon: <ImageIcon className="w-5 h-5" /> },
    { id: 'characters', label: 'Characters', icon: <Users className="w-5 h-5" /> },
    { id: 'environment', label: 'World', icon: <MapPin className="w-5 h-5" /> },
    { id: 'scene', label: 'Action', icon: <Clapperboard className="w-5 h-5" /> },
    { id: 'movement', label: 'Motion', icon: <Move className="w-5 h-5" /> },
    { id: 'camera', label: 'Camera', icon: <Camera className="w-5 h-5" /> },
    { id: 'lighting', label: 'Light', icon: <Sun className="w-5 h-5" /> },
    { id: 'audio', label: 'Audio', icon: <Music className="w-5 h-5" /> },
    { id: 'dialog', label: 'Dialog', icon: <MessageSquare className="w-5 h-5" /> }
  ];

  const handleInputChange = (section, field, value, index = null) => {
    if (index !== null) {
      const updatedChars = [...formData.characters];
      updatedChars[index][field] = value;
      setFormData({ ...formData, characters: updatedChars });
    } else if (field) {
      setFormData({
        ...formData,
        [section]: { ...formData[section], [field]: value }
      });
    } else {
      setFormData({ ...formData, [section]: value });
    }
  };

  const handleImageUpload = (e) => {
    const files = Array.from(e.target.files);
    if (images.length + files.length > 3) {
      alert("Maksimal 3 gambar referensi.");
      return;
    }
    
    files.forEach(file => {
      const reader = new FileReader();
      reader.onloadend = () => {
        setImages(prev => [...prev, reader.result]);
      };
      reader.readAsDataURL(file);
    });
  };

  const callGemini = async (prompt, imageData = null) => {
    let retries = 0;
    const maxRetries = 5;
    
    while (retries < maxRetries) {
      try {
        const payload = {
          contents: [{
            parts: [
              { text: prompt },
              ...(imageData ? imageData.map(img => ({
                inlineData: {
                  mimeType: "image/png",
                  data: img.split(',')[1]
                }
              })) : [])
            ]
          }],
          systemInstruction: {
            parts: [{ text: "You are a professional AI Video Prompt Engineer. Convert user inputs and visual references into professional English cinematic descriptions for the VEO3 model. Ensure consistency and precision." }]
          },
          generationConfig: {
            responseMimeType: "application/json",
            responseSchema: {
              type: "OBJECT",
              properties: {
                description: { type: "STRING" },
                analysis: { type: "STRING" },
                translated_fields: { type: "OBJECT" }
              }
            }
          }
        };

        const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(payload)
        });

        if (!response.ok) throw new Error("API request failed");
        
        const data = await response.json();
        return JSON.parse(data.candidates[0].content.parts[0].text);
      } catch (error) {
        retries++;
        await new Promise(res => setTimeout(res, Math.pow(2, retries) * 1000));
        if (retries === maxRetries) throw error;
      }
    }
  };

  const generateFinalPrompt = async () => {
    setIsGenerating(true);
    setStatusMessage("Menganalisis data dan merangkai JSON...");

    try {
      let imageDescription = "";
      if (images.length > 0) {
        setStatusMessage("Menganalisis gambar referensi...");
        const imageAnalysis = await callGemini("Analyze these images. Describe the primary characters, visual style, lighting, color palette, and environment in deep professional detail for an AI video prompt.", images);
        imageDescription = imageAnalysis.description;
      }

      setStatusMessage("Menyempurnakan deskripsi adegan...");
      const textAnalysis = await callGemini(`Convert these inputs into professional cinematic English:
        Environment: ${JSON.stringify(formData.environment)}
        Scene: ${JSON.stringify(formData.scene)}
        Movement: ${JSON.stringify(formData.movement)}
        Camera: ${JSON.stringify(formData.camera)}
        Lighting: ${JSON.stringify(formData.lighting)}
      `);

      const finalJson = {
        "metadata": {
          ...formData.metadata,
          "generated_at": new Date().toISOString()
        },
        "image_reference": {
          "enabled": images.length > 0,
          "description": imageDescription || "No reference images provided"
        },
        "characters": formData.characters.slice(0, formData.character_count).map((char, i) => ({
          "id": i + 1,
          ...char
        })),
        "world_setup": {
          "environment": formData.environment,
          "lighting_and_color": formData.lighting,
          "audio_settings": formData.audio
        },
        "cinematography": {
          "main_scene": formData.scene,
          "movement_logic": formData.movement,
          "camera_specs": formData.camera
        },
        "voice_layer": formData.dialog.enabled ? formData.dialog : null,
        "constraints": {
          "negative_prompt": "No camera shake, no teleport character, no sudden scene change, consistent character design, no flickering, no distorted limbs",
          "motion_bucket": 127,
          "cfg_scale": 7.5
        }
      };

      setOutputJson(finalJson);
      setActiveStep(steps.length); // Pindah ke layar hasil
    } catch (error) {
      console.error(error);
      alert("Gagal membuat prompt. Silakan coba lagi.");
    } finally {
      setIsGenerating(false);
      setStatusMessage("");
    }
  };

  const copyToClipboard = () => {
    const text = JSON.stringify(outputJson, null, 2);
    const textArea = document.createElement("textarea");
    textArea.value = text;
    document.body.appendChild(textArea);
    textArea.select();
    document.execCommand('copy');
    document.body.removeChild(textArea);
    alert("JSON berhasil disalin!");
  };

  const renderStep = () => {
    const current = steps[activeStep];
    if (!current) return null;

    switch (current.id) {
      case 'metadata':
        return (
          <div className="space-y-4 animate-in fade-in slide-in-from-bottom-4">
            <h3 className="text-xl font-bold flex items-center gap-2 text-indigo-400">
              <Video /> Metadata Video
            </h3>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
              <div className="space-y-1">
                <label className="text-sm text-slate-400">Model</label>
                <select 
                  className="w-full bg-slate-800 border border-slate-700 rounded-lg p-3 text-white focus:ring-2 focus:ring-indigo-500 transition-all"
                  value={formData.metadata.model}
                  onChange={(e) => handleInputChange('metadata', 'model', e.target.value)}
                >
                  <option value="veo-3">VEO-3 (Latest)</option>
                  <option value="veo-2">VEO-2 (Legacy)</option>
                </select>
              </div>
              <div className="space-y-1">
                <label className="text-sm text-slate-400">Duration</label>
                <input 
                  type="text"
                  placeholder="e.g. 7-8 seconds"
                  className="w-full bg-slate-800 border border-slate-700 rounded-lg p-3 text-white"
                  value={formData.metadata.duration}
                  onChange={(e) => handleInputChange('metadata', 'duration', e.target.value)}
                />
              </div>
              <div className="space-y-1">
                <label className="text-sm text-slate-400">Aspect Ratio</label>
                <div className="flex gap-2">
                  {['16:9', '9:16', '1:1'].map(ratio => (
                    <button 
                      key={ratio}
                      onClick={() => handleInputChange('metadata', 'aspect_ratio', ratio)}
                      className={`flex-1 p-2 rounded-lg border transition-all ${formData.metadata.aspect_ratio === ratio ? 'bg-indigo-600 border-indigo-400' : 'bg-slate-800 border-slate-700'}`}
                    >
                      {ratio}
                    </button>
                  ))}
                </div>
              </div>
              <div className="space-y-1">
                <label className="text-sm text-slate-400">Visual Style</label>
                <select 
                  className="w-full bg-slate-800 border border-slate-700 rounded-lg p-3 text-white"
                  value={formData.metadata.visual_style}
                  onChange={(e) => handleInputChange('metadata', 'visual_style', e.target.value)}
                >
                  <option value="cinematic">Cinematic</option>
                  <option value="anime">Anime</option>
                  <option value="hyper-realistic">Hyper-Realistic</option>
                  <option value="realistic">Realistic</option>
                  <option value="stylized-3d">Stylized 3D</option>
                </select>
              </div>
            </div>
          </div>
        );

      case 'images':
        return (
          <div className="space-y-4 animate-in fade-in slide-in-from-bottom-4">
            <h3 className="text-xl font-bold flex items-center gap-2 text-pink-400">
              <ImageIcon /> Referensi Gambar (Opsional)
            </h3>
            <p className="text-slate-400 text-sm">Upload hingga 3 gambar. Sistem akan mengekstrak detail visual secara otomatis.</p>
            
            <div className="grid grid-cols-3 gap-2">
              {images.map((img, idx) => (
                <div key={idx} className="relative group aspect-square rounded-lg overflow-hidden border border-slate-700 bg-slate-800">
                  <img src={img} className="w-full h-full object-cover" alt="Ref" />
                  <button 
                    onClick={() => setImages(images.filter((_, i) => i !== idx))}
                    className="absolute top-1 right-1 p-1 bg-red-500 rounded-full opacity-0 group-hover:opacity-100 transition-opacity"
                  >
                    <Trash2 className="w-3 h-3" />
                  </button>
                </div>
              ))}
              {images.length < 3 && (
                <label className="aspect-square border-2 border-dashed border-slate-700 rounded-lg flex flex-col items-center justify-center cursor-pointer hover:bg-slate-800 transition-colors">
                  <ImageIcon className="text-slate-500 mb-1" />
                  <span className="text-xs text-slate-500">Upload</span>
                  <input type="file" className="hidden" accept="image/*" onChange={handleImageUpload} multiple />
                </label>
              )}
            </div>
          </div>
        );

      case 'characters':
        return (
          <div className="space-y-6 animate-in fade-in slide-in-from-bottom-4">
            <div className="flex justify-between items-center">
              <h3 className="text-xl font-bold flex items-center gap-2 text-blue-400">
                <Users /> Karakter
              </h3>
              <div className="flex bg-slate-800 p-1 rounded-lg">
                <button 
                  onClick={() => handleInputChange('character_count', null, 1)}
                  className={`px-3 py-1 rounded-md text-sm transition-all ${formData.character_count === 1 ? 'bg-indigo-600' : ''}`}
                >1 Char</button>
                <button 
                  onClick={() => handleInputChange('character_count', null, 2)}
                  className={`px-3 py-1 rounded-md text-sm transition-all ${formData.character_count === 2 ? 'bg-indigo-600' : ''}`}
                >2 Chars</button>
              </div>
            </div>

            {[...Array(formData.character_count)].map((_, idx) => (
              <div key={idx} className="p-4 bg-slate-800/50 border border-slate-700 rounded-xl space-y-4">
                <div className="flex items-center gap-2 text-indigo-300 font-semibold mb-2">
                  <span className="w-6 h-6 rounded-full bg-indigo-500/20 flex items-center justify-center text-xs">
                    {idx + 1}
                  </span>
                  Karakter {idx + 1}
                </div>
                <div className="grid grid-cols-2 gap-3">
                  <input 
                    placeholder="Nama / Panggilan"
                    className="bg-slate-900 border border-slate-700 p-2 rounded-lg text-sm"
                    value={formData.characters[idx].name}
                    onChange={(e) => handleInputChange('characters', 'name', e.target.value, idx)}
                  />
                  <select 
                    className="bg-slate-900 border border-slate-700 p-2 rounded-lg text-sm"
                    value={formData.characters[idx].gender}
                    onChange={(e) => handleInputChange('characters', 'gender', e.target.value, idx)}
                  >
                    <option value="">Gender</option>
                    <option value="Male">Male</option>
                    <option value="Female">Female</option>
                    <option value="Non-binary">Non-binary</option>
                  </select>
                  <input 
                    placeholder="Usia (e.g. 25 years old)"
                    className="bg-slate-900 border border-slate-700 p-2 rounded-lg text-sm"
                    value={formData.characters[idx].age}
                    onChange={(e) => handleInputChange('characters', 'age', e.target.value, idx)}
                  />
                  <input 
                    placeholder="Warna/Gaya Rambut"
                    className="bg-slate-900 border border-slate-700 p-2 rounded-lg text-sm"
                    value={formData.characters[idx].hair}
                    onChange={(e) => handleInputChange('characters', 'hair', e.target.value, idx)}
                  />
                  <input 
                    placeholder="Pakaian (Detail)"
                    className="bg-slate-900 border border-slate-700 p-2 rounded-lg text-sm col-span-2"
                    value={formData.characters[idx].clothing}
                    onChange={(e) => handleInputChange('characters', 'clothing', e.target.value, idx)}
                  />
                  <input 
                    placeholder="Ekspresi"
                    className="bg-slate-900 border border-slate-700 p-2 rounded-lg text-sm"
                    value={formData.characters[idx].expression}
                    onChange={(e) => handleInputChange('characters', 'expression', e.target.value, idx)}
                  />
                  <input 
                    placeholder="Emosi Utama"
                    className="bg-slate-900 border border-slate-700 p-2 rounded-lg text-sm"
                    value={formData.characters[idx].emotion}
                    onChange={(e) => handleInputChange('characters', 'emotion', e.target.value, idx)}
                  />
                </div>
              </div>
            ))}
          </div>
        );

      case 'environment':
        return (
          <div className="space-y-4 animate-in fade-in slide-in-from-bottom-4">
            <h3 className="text-xl font-bold flex items-center gap-2 text-emerald-400">
              <MapPin /> Lingkungan & Atmosfer
            </h3>
            <div className="space-y-3">
              <input 
                placeholder="Lokasi (e.g. Cyberpunk alleyway, Forest at night)"
                className="w-full bg-slate-800 border border-slate-700 p-3 rounded-lg"
                value={formData.environment.location}
                onChange={(e) => handleInputChange('environment', 'location', e.target.value)}
              />
              <div className="grid grid-cols-2 gap-3">
                <input 
                  placeholder="Waktu (e.g. Golden hour)"
                  className="bg-slate-800 border border-slate-700 p-3 rounded-lg"
                  value={formData.environment.time}
                  onChange={(e) => handleInputChange('environment', 'time', e.target.value)}
                />
                <input 
                  placeholder="Cuaca (e.g. Heavy rain)"
                  className="bg-slate-800 border border-slate-700 p-3 rounded-lg"
                  value={formData.environment.weather}
                  onChange={(e) => handleInputChange('environment', 'weather', e.target.value)}
                />
              </div>
              <input 
                placeholder="Atmosfer / Vibe (e.g. Melancholic, Eerie)"
                className="w-full bg-slate-800 border border-slate-700 p-3 rounded-lg"
                value={formData.environment.atmosphere}
                onChange={(e) => handleInputChange('environment', 'atmosphere', e.target.value)}
              />
            </div>
          </div>
        );

      case 'scene':
        return (
          <div className="space-y-4 animate-in fade-in slide-in-from-bottom-4">
            <h3 className="text-xl font-bold flex items-center gap-2 text-amber-400">
              <Clapperboard /> Adegan Utama
            </h3>
            <textarea 
              placeholder="Apa yang terjadi dalam adegan ini?"
              className="w-full bg-slate-800 border border-slate-700 p-3 rounded-lg min-h-[100px]"
              value={formData.scene.description}
              onChange={(e) => handleInputChange('scene', 'description', e.target.value)}
            />
            <textarea 
              placeholder="Urutan aksi (e.g. Menoleh -> Berjalan -> Tersenyum)"
              className="w-full bg-slate-800 border border-slate-700 p-3 rounded-lg text-sm"
              value={formData.scene.action_sequence}
              onChange={(e) => handleInputChange('scene', 'action_sequence', e.target.value)}
            />
            {formData.character_count > 1 && (
              <input 
                placeholder="Interaksi antar karakter"
                className="w-full bg-slate-800 border border-slate-700 p-3 rounded-lg text-sm"
                value={formData.scene.interaction}
                onChange={(e) => handleInputChange('scene', 'interaction', e.target.value)}
              />
            )}
          </div>
        );

      case 'movement':
        return (
          <div className="space-y-4 animate-in fade-in slide-in-from-bottom-4">
            <h3 className="text-xl font-bold flex items-center gap-2 text-violet-400">
              <Move /> Pergerakan
            </h3>
            <div className="space-y-4">
              <div className="space-y-1">
                <label className="text-xs text-slate-500">Pergerakan Karakter 1</label>
                <input 
                  placeholder="e.g. Running towards camera"
                  className="w-full bg-slate-800 border border-slate-700 p-3 rounded-lg"
                  value={formData.movement.character_1}
                  onChange={(e) => handleInputChange('movement', 'character_1', e.target.value)}
                />
              </div>
              {formData.character_count > 1 && (
                <div className="space-y-1">
                  <label className="text-xs text-slate-500">Pergerakan Karakter 2</label>
                  <input 
                    placeholder="e.g. Standing still, watching"
                    className="w-full bg-slate-800 border border-slate-700 p-3 rounded-lg"
                    value={formData.movement.character_2}
                    onChange={(e) => handleInputChange('movement', 'character_2', e.target.value)}
                  />
                </div>
              )}
            </div>
          </div>
        );

      case 'camera':
        return (
          <div className="space-y-4 animate-in fade-in slide-in-from-bottom-4">
            <h3 className="text-xl font-bold flex items-center gap-2 text-orange-400">
              <Camera /> Kamera
            </h3>
            <div className="grid grid-cols-1 gap-3">
              <input 
                placeholder="Jenis Shot (e.g. Extreme Close-up, Wide Shot)"
                className="bg-slate-800 border border-slate-700 p-3 rounded-lg"
                value={formData.camera.shot_type}
                onChange={(e) => handleInputChange('camera', 'shot_type', e.target.value)}
              />
              <input 
                placeholder="Pergerakan Kamera (e.g. Slow Dolly In)"
                className="bg-slate-800 border border-slate-700 p-3 rounded-lg"
                value={formData.camera.movement}
                onChange={(e) => handleInputChange('camera', 'movement', e.target.value)}
              />
              <div className="grid grid-cols-2 gap-3">
                <input 
                  placeholder="Frame Awal (Start)"
                  className="bg-slate-800 border border-slate-700 p-3 rounded-lg text-sm"
                  value={formData.camera.start_frame}
                  onChange={(e) => handleInputChange('camera', 'start_frame', e.target.value)}
                />
                <input 
                  placeholder="Frame Akhir (End)"
                  className="bg-slate-800 border border-slate-700 p-3 rounded-lg text-sm"
                  value={formData.camera.end_frame}
                  onChange={(e) => handleInputChange('camera', 'end_frame', e.target.value)}
                />
              </div>
              <div className="space-y-1">
                <label className="text-xs text-slate-500">Stabilitas</label>
                <div className="flex gap-2">
                  {['stable', 'handheld', 'shaky'].map(s => (
                    <button 
                      key={s}
                      onClick={() => handleInputChange('camera', 'stability', s)}
                      className={`flex-1 p-2 rounded-lg border text-sm capitalize ${formData.camera.stability === s ? 'bg-indigo-600 border-indigo-400' : 'bg-slate-800 border-slate-700'}`}
                    >
                      {s}
                    </button>
                  ))}
                </div>
              </div>
            </div>
          </div>
        );

      case 'lighting':
        return (
          <div className="space-y-4 animate-in fade-in slide-in-from-bottom-4">
            <h3 className="text-xl font-bold flex items-center gap-2 text-yellow-400">
              <Sun /> Pencahayaan & Warna
            </h3>
            <div className="space-y-3">
              <input 
                placeholder="Jenis Cahaya (e.g. Neon, Volumetric Fog)"
                className="w-full bg-slate-800 border border-slate-700 p-3 rounded-lg"
                value={formData.lighting.type}
                onChange={(e) => handleInputChange('lighting', 'type', e.target.value)}
              />
              <input 
                placeholder="Arah Cahaya (e.g. Backlit, Rim lighting)"
                className="w-full bg-slate-800 border border-slate-700 p-3 rounded-lg"
                value={formData.lighting.direction}
                onChange={(e) => handleInputChange('lighting', 'direction', e.target.value)}
              />
              <input 
                placeholder="Tone Warna (e.g. Teal and Orange, Monochrome)"
                className="w-full bg-slate-800 border border-slate-700 p-3 rounded-lg"
                value={formData.lighting.tone}
                onChange={(e) => handleInputChange('lighting', 'tone', e.target.value)}
              />
            </div>
          </div>
        );

      case 'audio':
        return (
          <div className="space-y-4 animate-in fade-in slide-in-from-bottom-4">
            <h3 className="text-xl font-bold flex items-center gap-2 text-cyan-400">
              <Music /> Audio
            </h3>
            <div className="space-y-3">
              <input 
                placeholder="Ambient Sound (e.g. Busy city traffic, chirping birds)"
                className="w-full bg-slate-800 border border-slate-700 p-3 rounded-lg"
                value={formData.audio.ambient}
                onChange={(e) => handleInputChange('audio', 'ambient', e.target.value)}
              />
              <input 
                placeholder="Musik Latar (e.g. Lo-fi synthwave)"
                className="w-full bg-slate-800 border border-slate-700 p-3 rounded-lg"
                value={formData.audio.music}
                onChange={(e) => handleInputChange('audio', 'music', e.target.value)}
              />
              <div className="space-y-1">
                <label className="text-xs text-slate-500">Volume</label>
                <div className="flex gap-2">
                  {['low', 'balanced', 'loud'].map(v => (
                    <button 
                      key={v}
                      onClick={() => handleInputChange('audio', 'volume', v)}
                      className={`flex-1 p-2 rounded-lg border text-sm capitalize ${formData.audio.volume === v ? 'bg-indigo-600 border-indigo-400' : 'bg-slate-800 border-slate-700'}`}
                    >
                      {v}
                    </button>
                  ))}
                </div>
              </div>
            </div>
          </div>
        );

      case 'dialog':
        return (
          <div className="space-y-4 animate-in fade-in slide-in-from-bottom-4">
            <div className="flex justify-between items-center">
              <h3 className="text-xl font-bold flex items-center gap-2 text-rose-400">
                <MessageSquare /> Dialog (Opsional)
              </h3>
              <button 
                onClick={() => handleInputChange('dialog', 'enabled', !formData.dialog.enabled)}
                className={`w-12 h-6 rounded-full transition-colors relative ${formData.dialog.enabled ? 'bg-indigo-600' : 'bg-slate-700'}`}
              >
                <div className={`absolute top-1 w-4 h-4 bg-white rounded-full transition-all ${formData.dialog.enabled ? 'left-7' : 'left-1'}`} />
              </button>
            </div>
            
            {formData.dialog.enabled && (
              <div className="space-y-3 animate-in fade-in zoom-in-95 duration-200">
                <div className="grid grid-cols-2 gap-3">
                  <input 
                    placeholder="Bahasa"
                    className="bg-slate-800 border border-slate-700 p-3 rounded-lg text-sm"
                    value={formData.dialog.language}
                    onChange={(e) => handleInputChange('dialog', 'language', e.target.value)}
                  />
                  <input 
                    placeholder="Timestamp (e.g. 00:02)"
                    className="bg-slate-800 border border-slate-700 p-3 rounded-lg text-sm"
                    value={formData.dialog.timestamp}
                    onChange={(e) => handleInputChange('dialog', 'timestamp', e.target.value)}
                  />
                </div>
                <textarea 
                  placeholder="Isi teks dialog..."
                  className="w-full bg-slate-800 border border-slate-700 p-3 rounded-lg text-sm"
                  value={formData.dialog.text}
                  onChange={(e) => handleInputChange('dialog', 'text', e.target.value)}
                />
                <div className="grid grid-cols-2 gap-3">
                  <input 
                    placeholder="Nada Suara (e.g. Deep whisper)"
                    className="bg-slate-800 border border-slate-700 p-3 rounded-lg text-sm"
                    value={formData.dialog.voice_tone}
                    onChange={(e) => handleInputChange('dialog', 'voice_tone', e.target.value)}
                  />
                  <input 
                    placeholder="Emosi Suara"
                    className="bg-slate-800 border border-slate-700 p-3 rounded-lg text-sm"
                    value={formData.dialog.emotion}
                    onChange={(e) => handleInputChange('dialog', 'emotion', e.target.value)}
                  />
                </div>
              </div>
            )}
            {!formData.dialog.enabled && (
              <div className="p-8 border border-dashed border-slate-700 rounded-lg text-center text-slate-500 italic">
                Dialog dinonaktifkan untuk scene ini.
              </div>
            )}
          </div>
        );

      default:
        return null;
    }
  };

  const renderOutput = () => {
    return (
      <div className="space-y-4 animate-in zoom-in-95 duration-300">
        <div className="flex justify-between items-center">
          <h3 className="text-xl font-bold text-indigo-400 flex items-center gap-2">
            <Sparkles className="w-6 h-6" /> JSON VEO3 PROMPT
          </h3>
          <button 
            onClick={() => { setOutputJson(null); setActiveStep(0); }}
            className="text-slate-400 hover:text-white transition-colors flex items-center gap-1 text-sm"
          >
            <RefreshCw className="w-4 h-4" /> Reset
          </button>
        </div>
        
        <div className="relative group">
          <pre className="bg-slate-950 p-6 rounded-2xl border border-slate-800 text-xs text-indigo-300 overflow-x-auto max-h-[500px] scrollbar-hide">
            {JSON.stringify(outputJson, null, 2)}
          </pre>
          <button 
            onClick={copyToClipboard}
            className="absolute top-4 right-4 p-3 bg-indigo-600 hover:bg-indigo-500 rounded-xl shadow-lg transition-all active:scale-95"
          >
            <Copy className="w-5 h-5 text-white" />
          </button>
        </div>

        <div className="bg-slate-800/40 p-4 rounded-xl border border-slate-700 space-y-2">
          <div className="flex items-center gap-2 text-sm text-slate-300">
            <Ban className="w-4 h-4 text-red-400" />
            <span className="font-semibold">Negative Prompt Otomatis:</span>
          </div>
          <p className="text-xs text-slate-500 leading-relaxed">
            No camera shake, no teleport character, no sudden scene change, consistent character design, no flickering, no distorted limbs, no text overlays, no watermarks.
          </p>
        </div>

        <button 
          onClick={() => setActiveStep(0)}
          className="w-full py-4 bg-slate-800 hover:bg-slate-700 rounded-xl font-bold transition-all"
        >
          Edit Prompt
        </button>
      </div>
    );
  };

  return (
    <div className="min-h-screen bg-[#0f1117] text-slate-100 p-4 md:p-8 font-sans">
      <div className="max-w-2xl mx-auto space-y-6">
        
        {/* Header */}
        <header className="flex items-center justify-between mb-8">
          <div className="flex items-center gap-3">
            <div className="p-3 bg-indigo-600 rounded-2xl shadow-xl shadow-indigo-900/20">
              <Video className="w-6 h-6 text-white" />
            </div>
            <div>
              <h1 className="text-xl font-black tracking-tight leading-none">VEO3 VIDEO</h1>
              <p className="text-xs font-medium text-slate-500 tracking-widest uppercase">Prompt Generator</p>
            </div>
          </div>
          <div className="text-[10px] px-2 py-1 bg-slate-800 rounded border border-slate-700 text-slate-400">
            v1.0 PRO
          </div>
        </header>

        {/* Progress Bar */}
        {activeStep < steps.length && (
          <div className="w-full h-1 bg-slate-800 rounded-full mb-8 overflow-hidden">
            <div 
              className="h-full bg-gradient-to-r from-indigo-500 to-pink-500 transition-all duration-500"
              style={{ width: `${((activeStep + 1) / steps.length) * 100}%` }}
            />
          </div>
        )}

        {/* Main Interface */}
        <div className="bg-slate-900/50 backdrop-blur-xl border border-slate-800 p-6 md:p-8 rounded-[2.5rem] shadow-2xl relative overflow-hidden">
          
          {/* Decorative background elements */}
          <div className="absolute top-0 right-0 w-64 h-64 bg-indigo-600/5 blur-[120px] pointer-events-none" />
          <div className="absolute bottom-0 left-0 w-64 h-64 bg-pink-600/5 blur-[120px] pointer-events-none" />

          {isGenerating ? (
            <div className="py-20 flex flex-col items-center justify-center space-y-4 animate-pulse">
              <div className="relative">
                <Loader2 className="w-12 h-12 text-indigo-500 animate-spin" />
                <Sparkles className="w-6 h-6 text-pink-500 absolute -top-2 -right-2 animate-bounce" />
              </div>
              <p className="text-lg font-medium text-indigo-300">{statusMessage}</p>
              <p className="text-xs text-slate-500">Mohon tunggu sebentar...</p>
            </div>
          ) : (
            <>
              {activeStep < steps.length ? (
                <>
                  <div className="min-h-[400px]">
                    {renderStep()}
                  </div>

                  {/* Navigation Footer */}
                  <div className="mt-12 flex items-center justify-between gap-4">
                    <button 
                      onClick={() => setActiveStep(prev => Math.max(0, prev - 1))}
                      disabled={activeStep === 0}
                      className={`flex items-center gap-2 px-6 py-3 rounded-2xl font-semibold transition-all ${activeStep === 0 ? 'opacity-30 cursor-not-allowed' : 'hover:bg-slate-800 text-slate-400'}`}
                    >
                      <ChevronLeft className="w-5 h-5" /> Back
                    </button>

                    {activeStep === steps.length - 1 ? (
                      <button 
                        onClick={generateFinalPrompt}
                        className="flex-1 bg-gradient-to-r from-indigo-600 to-indigo-500 hover:from-indigo-500 hover:to-indigo-400 py-4 rounded-2xl font-bold flex items-center justify-center gap-2 shadow-lg shadow-indigo-900/20 active:scale-[0.98] transition-all"
                      >
                        Generate Prompt <Sparkles className="w-5 h-5" />
                      </button>
                    ) : (
                      <button 
                        onClick={() => setActiveStep(prev => prev + 1)}
                        className="flex items-center gap-2 px-8 py-3 bg-slate-800 hover:bg-slate-700 rounded-2xl font-bold transition-all"
                      >
                        Next <ChevronRight className="w-5 h-5" />
                      </button>
                    )}
                  </div>
                </>
              ) : (
                renderOutput()
              )}
            </>
          )}
        </div>

        {/* Step Icons Indicator (Mobile friendly horizontal scroll) */}
        {activeStep < steps.length && (
          <div className="flex gap-2 overflow-x-auto pb-4 scrollbar-hide px-2">
            {steps.map((step, idx) => (
              <button
                key={step.id}
                onClick={() => setActiveStep(idx)}
                className={`flex-shrink-0 w-10 h-10 rounded-xl flex items-center justify-center transition-all ${activeStep === idx ? 'bg-indigo-600 text-white shadow-lg shadow-indigo-900/30' : 'bg-slate-800/50 text-slate-500 hover:bg-slate-800'}`}
              >
                {step.icon}
              </button>
            ))}
          </div>
        )}

        <footer className="text-center text-[10px] text-slate-600 font-medium pt-4">
          PROFESSIONAL VIDEO AI WORKFLOW &copy; 2025 • OPTIMIZED FOR VEO3
        </footer>
      </div>
      
      <style>{`
        .scrollbar-hide::-webkit-scrollbar {
          display: none;
        }
        .scrollbar-hide {
          -ms-overflow-style: none;
          scrollbar-width: none;
        }
      `}</style>
    </div>
  );
};

export default App;

