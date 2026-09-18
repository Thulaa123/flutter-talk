# Genkit Dart in 5 minutes — the smallest possible project

A step-by-step guide to a Genkit backend with **one flow, one Gemini call, served over HTTP**. The finished project lives in [`genkit-hello/`](genkit-hello/) (one file, ~30 lines) and was run exactly as written below.

**What you need:** Dart 3.13+ (comes with Flutter) and a free Gemini API key from https://aistudio.google.com/apikey.

---

## Step 1 — Create a Dart console project

```bash
dart create -t console genkit-hello
cd genkit-hello
```

## Step 2 — Add Genkit and the Gemini plugin

```bash
dart pub add 'genkit:^0.16.1' 'genkit_google_genai:^0.3.1' 'genkit_shelf:^0.1.13'
```

| Package | Why |
|---|---|
| `genkit` | the framework: `Genkit`, `defineFlow`, `generate`, tools, agents |
| `genkit_google_genai` | the Gemini plugin: `googleAI()` and `googleAI.gemini('…')` |
| `genkit_shelf` | serves flows over HTTP with `startFlowServer` |

> The `^0.16.1` pin matters: `genkit` 0.17 (Sept 2026) changed the client API. Everything in this repo is on 0.16.1.

## Step 3 — Write the whole app

Replace `bin/genkit_hello.dart` with:

```dart
import 'package:genkit/genkit.dart';
import 'package:genkit_google_genai/genkit_google_genai.dart';
import 'package:genkit_shelf/genkit_shelf.dart';

void main() async {
  // 1. Genkit + the Gemini plugin. The plugin reads GEMINI_API_KEY itself.
  final ai = Genkit(plugins: [googleAI()]);

  // 2. A flow: a typed function with a name. Input: city. Output: text.
  final suggestActivity = ai.defineFlow(
    name: 'suggestActivity',
    inputSchema: .string(),
    outputSchema: .string(),
    fn: (String city, _) async {
      final response = await ai.generate(
        model: googleAI.gemini('gemini-flash-latest'),
        prompt: 'Suggest one fun thing to do in $city on a Friday night. '
            'One sentence.',
      );
      return response.text;
    },
  );

  // 3. Serve it. POST /suggestActivity  {"data": "Colombo"}  →  {"result": "..."}
  await startFlowServer(flows: [suggestActivity], port: 3400);
}
```

Three ideas, three numbered comments:

1. **`Genkit(plugins: [...])`** is the app object. Plugins add models (Gemini here), middleware, catalogs.
2. **A flow** is a named, typed function. Genkit gives it tracing, a schema, and an HTTP endpoint for free. Inside it, `ai.generate(...)` is the unified model call.
3. **`startFlowServer`** mounts every flow at `POST /<flowName>`.

## Step 4 — Run it

```bash
export GEMINI_API_KEY=AIza...        # never commit this
dart run bin/genkit_hello.dart
# Flow server running on http://localhost:3400
```

## Step 5 — Call it

```bash
curl -X POST localhost:3400/suggestActivity \
  -H 'Content-Type: application/json' \
  -d '{"data": "Colombo"}'
```

```json
{"result":"Enjoy the vibrant nightlife at Park Street Mews by hopping between lively bars, restaurants, and live music spots set along a charming, fairy-lit cobblestone street."}
```

That envelope — `{"data": input}` in, `{"result": output}` out — is all a client needs to know. Part 2 wraps it in Flutter.

---

# Part 2 — Call the flow from Flutter

Finished app: [`flutter-hello/`](flutter-hello/) (one `main.dart`, ~70 lines). Keep the server from Step 4 running.

## Step 6 — Create the Flutter app and add the Genkit client

```bash
flutter create --platforms=macos,web flutter-hello
cd flutter-hello
flutter pub add 'genkit:^0.16.1'
```

The same `genkit` package works on the client: import only `package:genkit/client.dart`. It is browser-safe and pulls in nothing server-side.

**macOS only:** the Flutter template sandboxes the app, so allow outgoing connections. Add this to `macos/Runner/DebugProfile.entitlements` **and** `macos/Runner/Release.entitlements`:

```xml
<key>com.apple.security.network.client</key>
<true/>
```

## Step 7 — Write the whole app

Replace `lib/main.dart` with:

```dart
import 'package:flutter/material.dart';
import 'package:genkit/client.dart';

// 1. A typed handle on the remote flow: String in, String out.
final suggestActivity = defineRemoteAction(
  url: 'http://localhost:3400/suggestActivity',
  inputSchema: .string(),
  outputSchema: .string(),
);

void main() => runApp(const MaterialApp(home: HelloGenkit()));

class HelloGenkit extends StatefulWidget {
  const HelloGenkit({super.key});
  @override
  State<HelloGenkit> createState() => _HelloGenkitState();
}

class _HelloGenkitState extends State<HelloGenkit> {
  final city = TextEditingController(text: 'Colombo');
  String result = '';
  bool busy = false;

  // 2. Call the flow like a function. Errors arrive as GenkitException.
  Future<void> ask() async {
    setState(() => busy = true);
    try {
      final text = await suggestActivity(input: city.text);
      setState(() => result = text);
    } on GenkitException catch (e) {
      setState(() => result = 'Error: ${e.message}');
    } finally {
      setState(() => busy = false);
    }
  }

  // 3. Plain Flutter from here on.
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Flutter → Genkit')),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          children: [
            TextField(
              controller: city,
              decoration: const InputDecoration(labelText: 'City'),
              onSubmitted: (_) => ask(),
            ),
            const SizedBox(height: 16),
            FilledButton(
              onPressed: busy ? null : ask,
              child: Text(busy ? 'Asking Gemini…' : 'Suggest something ✨'),
            ),
            const SizedBox(height: 24),
            Text(result, style: Theme.of(context).textTheme.titleMedium),
          ],
        ),
      ),
    );
  }
}
```

Three ideas again:

1. **`defineRemoteAction`** is the client twin of `defineFlow`: same name, same schemas, one URL. It builds the `{"data": …}` request, parses `{"result": …}`, and validates against the schema.
2. **Calling it is calling a function.** `await suggestActivity(input: 'Colombo')` returns a `String`. Server or network failures come back as `GenkitException` with a readable `message`.
3. **No key in the app.** The key lives with the server from Part 1. Flutter only knows a URL.

## Step 8 — Run it

```bash
flutter run -d macos      # or: flutter run -d chrome
```

Type a city, press **Suggest something ✨**, read the sentence Gemini wrote. (Verified: "Kandy" → *"Head up to the rooftop of Slightly Chilled Lounge for drinks, great music, and vibrant nightlife overlooking the illuminated Kandy Lake."*)

Android emulator: change the URL to `http://10.0.2.2:3400`. Chrome: the server from Part 1 already sends CORS headers when you pass `cors: {'origin': '*'}` to `startFlowServer` — add that if the browser blocks the call.

### From here to Friday AI

| You just did | Friday AI does |
|---|---|
| `.string()` in and out | `@Schema()` classes on both sides → typed `ActivityRecommendation` (`app/lib/models.dart`) |
| one `defineRemoteAction` | one per flow (`app/lib/friday_api.dart`) |
| a flow that returns text | a flow with `outputSchema`, a tool, and an agent with sessions |
| plain `Text(result)` | a chat transcript, and in GenUI mode the agent draws the widgets (`genui-mini/`) |

---

## Where to go next (each is a few lines, all used in this repo)

**Structured output instead of text.** Declare a schema class and ask Gemini to fill it:

```dart
@Schema()                       // import 'package:schemantic/schemantic.dart';
abstract class $Activity {     // dart run build_runner build → Activity + Activity.$schema
  String get title;
  double get estimatedCost;
}

final res = await ai.generate(
  model: googleAI.gemini('gemini-flash-latest'),
  prompt: 'One activity in $city under Rs. 5000',
  outputSchema: Activity.$schema,   // Gemini must return this shape
);
final Activity a = res.output!;   // typed, no JSON parsing
```
Needs `dart pub add schemantic dev:schemantic_builder dev:build_runner`. See `genkit-server/lib/schemas.dart` and `flows.dart`.

**A tool Gemini can call.** A name, a description the model reads, a typed input, a Dart function:

```dart
final searchEvents = ai.defineTool(
  name: 'searchEvents',
  description: 'Events on tonight in a city, under maxPrice.',
  inputSchema: EventSearchInput.$schema,
  fn: (input, _) async => .response(lookUp(input.location, input.maxPrice)),
);

await ai.generate(model: ..., prompt: ..., toolNames: ['searchEvents']);
```
See `genkit-server/lib/tools.dart`.

**A conversational agent (sessions + streaming).**

```dart
final agent = ai.defineAgent(
  name: 'plainAgent',
  model: googleAI.gemini('gemini-flash-latest'),
  system: 'You help people plan a Friday night.',
  tools: [searchEvents],
  store: InMemorySessionStore(),
);
// serve: shelfHandler(agent.action) + /getSnapshot + /abort  → see genkit-server/bin/server.dart
// client: remoteAgent(url).chat().sendStream(text: '...')     → see app/lib/chat/chat_controller.dart
```

**Generative UI.** Add one middleware and a catalog: `use: [a2ui(catalog: ...)]` from `genkit_a2ui`. See `genkit-server/lib/agents.dart` and the one-file Flutter client in `genui-mini/`.

**Real web data.** Turn on Google Search grounding for a call: `config: GeminiOptions(googleSearch: GoogleSearch())`. See `genkit-server/lib/web_events.dart`.

**Developer UI.** `curl -sL cli.genkit.dev | bash` then `genkit start -- dart run bin/genkit_hello.dart` opens a local UI to run flows and inspect traces.
