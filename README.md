<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>VEO3 Video Prompt Generator</title>
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/lucide@latest"></script>
    <link href="https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;500;600;700;800&display=swap" rel="stylesheet">
    <style>
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #0f1117; color: #f8fafc; }
        .scrollbar-hide::-webkit-scrollbar { display: none; }
        .glass-card { background: rgba(30, 41, 59, 0.5); backdrop-filter: blur(12px); border: 1px solid rgba(255, 255, 255, 0.1); }
        @keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
        .animate-fade { animation: fadeIn 0.4s ease-out forwards; }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect, useCallback } = React;
        const apiKey = ""; // Kunci API akan diisi secara otomatis oleh lingkungan eksekusi

        // --- Ikon Komponen (Manual SVG untuk kemudahan akses) ---
        const Icon = ({ name, className = "w-5 h-5" }) => {
            const [svg, setSvg] = useState("");
            useEffect(() => {
                if (window.lucide) {
                    const iconNode = window.lucide.icons[name];
                    if (iconNode) {
                        const svgString = `<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="${className}">${iconNode[1].map(child => `<${child[0]} ${Object.entries(child[1]).map(([k, v]) => `${k}="${v}"`).join(' ')} />`).join('')}</svg>`;
                        setSvg(svgString);
                    }
                }
            }, [name]);
            return <span dangerouslySetInnerHTML={{ __html: svg }} />;
        };

        const App = () => {
            const [activeStep, setActiveStep] = useState(0);
            const [isGenerating, setIsGenerating] = useState(false);
            const [outputJson, setOutputJson] = useState(null);
            const [images, setImages] = useState([]);
            const [statusMessage, setStatusMessage] = useState("");

            const [formData, setFormData] = useState({
                metadata: { model: 'veo-3', duration: '7-8 seconds', aspect_ratio: '16:9', visual_style: 'cinematic', quality: 'high' },
                character_count: 1,
                characters: [
                    { name: '', gender: '', age: '', hair: '', eyes: '', clothing: '', expression: '', emotion: '' },
                    { name: '', gender: '', age: '', hair: '', eyes: '', clothing: '', expression: '', emotion: '' }
                ],
                environment: { location: '', time: '', weather: '', atmosphere: '' },
                scene: { description: '', action_sequence: '', interaction: '' },
                movement: { character_1: '', character_2: '' },
                camera: { shot_type: '', movement: '', start_frame: '', end_frame: '', stability: 'stable' },
                lighting: { type: '', direction: '', tone: '' },
                audio: { ambient: '', music: '', volume: 'balanced' },
                dialog: { enabled: false, language: 'English', timestamp: '00:01', text: '', voice_tone: '', emotion: '' }
            });

            const steps = [
                { id: 'metadata', label: 'Meta', icon: 'Video' },
                { id: 'images', label: 'Ref', icon: 'Image' },
                { id: 'characters', label: 'Chars', icon: 'Users' },
                { id: 'environment', label: 'World', icon: 'MapPin' },
                { id: 'scene', label: 'Action', icon: 'Clapperboard' },
                { id: 'camera', label: 'Cam', icon: 'Camera' },
                { id: 'audio', label: 'Audio', icon: 'Music' },
                { id: 'dialog', label: 'Talk', icon: 'MessageSquare' }
            ];

            const handleInputChange = (section, field, value, index = null) => {
                if (index !== null) {
                    const updatedChars = [...formData.characters];
                    updatedChars[index][field] = value;
                    setFormData({ ...formData, characters: updatedChars });
                } else if (field) {
                    setFormData({ ...formData, [section]: { ...formData[section], [field]: value } });
                } else {
                    setFormData({ ...formData, [section]: value });
                }
            };

            const handleImageUpload = (e) => {
                const files = Array.from(e.target.files);
                if (images.length + files.length > 3) return;
                files.forEach(file => {
                    const reader = new FileReader();
                    reader.onloadend = () => setImages(prev => [...prev, reader.result]);
                    reader.readAsDataURL(file);
                });
            };

            const callGemini = async (prompt, imageData = null) => {
                let retries = 0;
                while (retries < 5) {
                    try {
                        const payload = {
                            contents: [{
                                parts: [{ text: prompt }, ...(imageData ? imageData.map(img => ({ inlineData: { mimeType: "image/png", data: img.split(',')[1] } })) : [])]
                            }],
                            systemInstruction: { parts: [{ text: "You are a professional VEO3 Video Prompt Engineer. Use professional cinematic English for all technical descriptions. Response must be JSON." }] },
                            generationConfig: { responseMimeType: "application/json" }
                        };
                        const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
                            method: 'POST',
                            headers: { 'Content-Type': 'application/json' },
                            body: JSON.stringify(payload)
                        });
                        if (!response.ok) throw new Error("API fail");
                        const data = await response.json();
                        return JSON.parse(data.candidates[0].content.parts[0].text);
                    } catch (e) {
                        retries++;
                        await new Promise(r => setTimeout(r, Math.pow(2, retries) * 1000));
                    }
                }
            };

            const generate = async () => {
                setIsGenerating(true);
                setStatusMessage("Menganalisis data...");
                try {
                    let imageDesc = "";
                    if (images.length > 0) {
                        setStatusMessage("Menganalisis Gambar...");
                        const res = await callGemini("Analyze characters and style.", images);
                        imageDesc = res.description || "Visual defined by reference";
                    }
                    setStatusMessage("Menyusun JSON...");
                    const final = {
                        metadata: formData.metadata,
                        image_reference: { enabled: images.length > 0, description: imageDesc },
                        characters: formData.characters.slice(0, formData.character_count),
                        cinematography: { scene: formData.scene, camera: formData.camera },
                        constraints: "Consistent character, no scene change"
                    };
                    setOutputJson(final);
                    setActiveStep(steps.length);
                } catch (e) {
                    console.error(e);
                } finally {
                    setIsGenerating(false);
                }
            };

            const copy = () => {
                document.execCommand('copy', false, JSON.stringify(outputJson, null, 2));
                const dummy = document.createElement("textarea");
                document.body.appendChild(dummy);
                dummy.value = JSON.stringify(outputJson, null, 2);
                dummy.select();
                document.execCommand("copy");
                document.body.removeChild(dummy);
            };

            return (
                <div className="max-w-xl mx-auto px-4 py-8">
                    <header className="flex items-center gap-3 mb-8">
                        <div className="p-3 bg-indigo-600 rounded-2xl"><Icon name="Video" className="text-white" /></div>
                        <div>
                            <h1 className="text-xl font-bold leading-none">VEO3 PROMPT</h1>
                            <p className="text-[10px] tracking-widest text-slate-500 uppercase font-bold">Generator Tool</p>
                        </div>
                    </header>

                    <div className="glass-card rounded-[2rem] p-6 shadow-2xl relative overflow-hidden">
                        {isGenerating ? (
                            <div className="py-20 text-center">
                                <div className="w-12 h-12 border-4 border-indigo-500 border-t-transparent rounded-full animate-spin mx-auto mb-4"></div>
                                <p className="text-indigo-300 font-medium">{statusMessage}</p>
                            </div>
                        ) : activeStep < steps.length ? (
                            <div className="animate-fade">
                                <h2 className="text-lg font-bold mb-4 flex items-center gap-2">
                                    <Icon name={steps[activeStep].icon} /> {steps[activeStep].label}
                                </h2>
                                
                                {activeStep === 0 && (
                                    <div className="grid grid-cols-2 gap-3">
                                        <select className="bg-slate-800 p-3 rounded-xl border border-slate-700" value={formData.metadata.aspect_ratio} onChange={e => handleInputChange('metadata', 'aspect_ratio', e.target.value)}>
                                            <option value="16:9">16:9 Landscape</option>
                                            <option value="9:16">9:16 Portrait</option>
                                            <option value="1:1">1:1 Square</option>
                                        </select>
                                        <select className="bg-slate-800 p-3 rounded-xl border border-slate-700" value={formData.metadata.visual_style} onChange={e => handleInputChange('metadata', 'visual_style', e.target.value)}>
                                            <option value="cinematic">Cinematic</option>
                                            <option value="anime">Anime</option>
                                            <option value="realistic">Realistic</option>
                                        </select>
                                    </div>
                                )}

                                {activeStep === 1 && (
                                    <div className="grid grid-cols-3 gap-2">
                                        {images.map((img, i) => (
                                            <img key={i} src={img} className="aspect-square object-cover rounded-lg border border-slate-700" />
                                        ))}
                                        {images.length < 3 && (
                                            <label className="aspect-square border-2 border-dashed border-slate-700 rounded-lg flex items-center justify-center cursor-pointer">
                                                <Icon name="Plus" />
                                                <input type="file" className="hidden" onChange={handleImageUpload} />
                                            </label>
                                        )}
                                    </div>
                                )}

                                {activeStep > 1 && (
                                    <div className="space-y-3">
                                        <textarea 
                                            placeholder="Masukkan detail deskripsi di sini..."
                                            className="w-full bg-slate-800 p-4 rounded-xl border border-slate-700 min-h-[150px]"
                                            onChange={e => {
                                                const section = steps[activeStep].id;
                                                handleInputChange(section, 'description', e.target.value);
                                            }}
                                        />
                                    </div>
                                )}

                                <div className="mt-8 flex justify-between">
                                    <button onClick={() => setActiveStep(s => s - 1)} disabled={activeStep === 0} className="px-6 py-3 text-slate-500 font-bold">Kembali</button>
                                    {activeStep === steps.length - 1 ? (
                                        <button onClick={generate} className="bg-indigo-600 px-8 py-3 rounded-xl font-bold shadow-lg">Generate</button>
                                    ) : (
                                        <button onClick={() => setActiveStep(s => s + 1)} className="bg-slate-800 px-8 py-3 rounded-xl font-bold border border-slate-700">Lanjut</button>
                                    )}
                                </div>
                            </div>
                        ) : (
                            <div className="animate-fade">
                                <div className="flex justify-between items-center mb-4">
                                    <h2 className="font-bold text-indigo-400">JSON GENERATED</h2>
                                    <button onClick={copy} className="bg-indigo-600 p-2 rounded-lg"><Icon name="Copy" /></button>
                                </div>
                                <pre className="bg-slate-950 p-4 rounded-xl text-[10px] overflow-auto max-h-[400px]">
                                    {JSON.stringify(outputJson, null, 2)}
                                </pre>
                                <button onClick={() => setActiveStep(0)} className="w-full mt-4 py-3 bg-slate-800 rounded-xl font-bold">Mulai Baru</button>
                            </div>
                        )}
                    </div>

                    <div className="mt-6 flex justify-center gap-2 overflow-x-auto scrollbar-hide py-2">
                        {steps.map((s, i) => (
                            <div key={i} className={`w-2 h-2 rounded-full transition-all ${activeStep === i ? 'w-6 bg-indigo-500' : 'bg-slate-700'}`} />
                        ))}
                    </div>
                </div>
            );
        };

            const root = ReactDOM.createRoot(document.getElementById('root'));
            root.render(<App />);
    </script>
</body>
</html>

