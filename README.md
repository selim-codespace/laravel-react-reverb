# Laravel + Reverb + React Chat App

A real-time chat application built with Laravel Reverb for the backend and React for the frontend.

## 📋 Table of Contents

- [Features](#features)
- [Prerequisites](#prerequisites)
- [Backend Setup (Laravel)](#backend-setup-laravel)
- [Frontend Setup (React)](#frontend-setup-react)
- [Running the Application](#running-the-application)
- [Project Structure](#project-structure)

## ✨ Features

- Real-time messaging using Laravel Reverb
- Private chat channels
- Laravel Sanctum authentication
- WebSocket connections
- Message persistence in database

## 🔧 Prerequisites

- PHP >= 8.1
- Composer
- Node.js >= 16.x
- npm or yarn
- MySQL or PostgreSQL

## 🚀 Backend Setup (Laravel)

### 1. Install Laravel Reverb

```bash
composer require laravel/reverb
php artisan vendor:publish --tag=reverb-config
```

### 2. Configure Broadcasting Channels

Create or update `routes/channels.php`:

```php
<?php
use Illuminate\Support\Facades\Broadcast;

Broadcast::channel('chat.{receiverId}', function ($user, $receiverId) {
    return (int) $user->id === (int) $receiverId;
});
```

### 3. Set Up API Routes

Update `routes/api.php`:

```php
<?php
use App\Events\MessageSent;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\AuthController;
use App\Http\Controllers\MessageController;
use Illuminate\Support\Facades\Broadcast;

Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');

Broadcast::routes(['middleware' => ['auth:sanctum']]);

Route::post('/auth/login', [AuthController::class, 'login']);

Route::middleware('auth:sanctum')->group(function () {
    Route::get('/auth/me', [AuthController::class, 'me']);
    Route::post('/auth/logout', [AuthController::class, 'logout']);
    Route::post('/message/send', [MessageController::class, 'sendmsg']);
    Route::get('/messages/{user}', [MessageController::class, 'getMessages']);
});
```

### 4. Create the MessageSent Event

Create `app/Events/MessageSent.php`:

```php
<?php
namespace App\Events;

use App\Models\Message;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;
use Illuminate\Broadcasting\InteractsWithSockets;

class MessageSent implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public $message;

    public function __construct(Message $message)
    {
        $this->message = $message;
    }

    public function broadcastOn()
    {
        return new PrivateChannel('chat.' . $this->message->receiver_id);
    }

    public function broadcastWith()
    {
        return [
            'id' => $this->message->id,
            'sender_id' => $this->message->sender_id,
            'receiver_id' => $this->message->receiver_id,
            'message' => $this->message->message,
            'created_at' => $this->message->created_at,
        ];
    }
}
```

### 5. Implement Message Controller

Add to `app/Http/Controllers/MessageController.php`:

```php
public function sendmsg(Request $request)
{
    $request->validate([
        'receiver_id' => 'required|exists:users,id',
        'message' => 'required|string',
    ]);

    $message = Message::create([
        'sender_id' => Auth::id(),
        'receiver_id' => $request->receiver_id,
        'message' => $request->message,
    ]);

    broadcast(new MessageSent($message))->toOthers();

    return response()->json($message);
}
```

### 6. Configure CORS

Update `config/cors.php`:

```php
<?php
return [
    'paths' => ['api/*', 'sanctum/csrf-cookie', 'broadcasting/auth'],
    'allowed_methods' => ['*'],
    'allowed_origins' => [
        'http://localhost:5173',
        'http://127.0.0.1:5173',
    ],
    'allowed_headers' => ['*'],
    'supports_credentials' => true,
];
```

### 7. Configure Sanctum

Update `config/sanctum.php`:

```php
'stateful' => explode(',', env('SANCTUM_STATEFUL_DOMAINS', sprintf(
    '%s%s',
    'localhost,localhost:3000,127.0.0.1,127.0.0.1:8000,::1,localhost:5173',
    Sanctum::currentApplicationUrlWithPort(),
))),
```

### 8. Environment Variables

Add to your `.env` file:

```env
BROADCAST_DRIVER=reverb
REVERB_APP_ID=your-app-id
REVERB_APP_KEY=your-app-key
REVERB_APP_SECRET=your-app-secret
REVERB_HOST=localhost
REVERB_PORT=8080
REVERB_SCHEME=http
```

## ⚛️ Frontend Setup (React)

### 1. Install Dependencies

```bash
npm install axios laravel-echo pusher-js
```

### 2. Environment Variables

Create `.env` file in your React project:

```env
VITE_REVERB_APP_KEY=your-app-key
VITE_REVERB_HOST=localhost
VITE_REVERB_PORT=8080
VITE_API_URL=http://127.0.0.1:8000/api
```

### 3. Configure Laravel Echo

Add this to your React component:

```javascript
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

window.Pusher = Pusher;

useEffect(() => {
  const storedUser = localStorage.getItem("user");
  if (!storedUser) return;

  const userData = JSON.parse(storedUser);
  setUser(userData);
  const token = localStorage.getItem("authToken");

  const echoInstance = new Echo({
    broadcaster: "reverb",
    key: import.meta.env.VITE_REVERB_APP_KEY,
    wsHost: import.meta.env.VITE_REVERB_HOST || "localhost",
    wsPort: import.meta.env.VITE_REVERB_PORT || 8080,
    wssPort: import.meta.env.VITE_REVERB_PORT || 443,
    forceTLS: false,
    enabledTransports: ["ws", "wss"],
    auth: {
      headers: { 
        Authorization: `Bearer ${token}`, 
        Accept: "application/json" 
      },
    },
    authEndpoint: "http://127.0.0.1:8000/api/broadcasting/auth",
  });

  const channelName = `chat.${userData.id}`;
  echoInstance.private(channelName).listen("MessageSent", (e) => {
    setMessages((prev) => [...prev, e]);
  });

  echoInstance.connector.pusher.connection.bind("connected", () =>
    console.log("✅ Connected to Reverb")
  );
  echoInstance.connector.pusher.connection.bind("error", (err) =>
    console.error("❌ Reverb error:", err)
  );

  setEcho(echoInstance);

  return () => {
    echoInstance.leave(channelName);
    echoInstance.disconnect();
  };
}, []);
```

## 🏃 Running the Application

### Start Laravel Backend

```bash
# Run migrations
php artisan migrate

# Start Laravel development server
php artisan serve

# Start Reverb WebSocket server (in a separate terminal)
php artisan reverb:start
```

### Start React Frontend

```bash
# Install dependencies (if not already done)
npm install

# Start development server
npm run dev
```

The application will be available at:
- Frontend: `http://localhost:5173`
- Backend API: `http://127.0.0.1:8000`
- WebSocket: `ws://localhost:8080`

## 📁 Project Structure

```
laravel-reverb-chat/
├── backend/
│   ├── app/
│   │   ├── Events/
│   │   │   └── MessageSent.php
│   │   ├── Http/
│   │   │   └── Controllers/
│   │   │       ├── AuthController.php
│   │   │       └── MessageController.php
│   │   └── Models/
│   │       └── Message.php
│   ├── routes/
│   │   ├── api.php
│   │   └── channels.php
│   └── config/
│       ├── cors.php
│       └── sanctum.php
└── frontend/
    ├── src/
    │   ├── components/
    │   └── App.jsx
    └── package.json
```

## 📝 How It Works

1. **Authentication**: Users authenticate via Laravel Sanctum, receiving a token stored in localStorage
2. **WebSocket Connection**: React connects to Laravel Reverb using Laravel Echo and Pusher
3. **Private Channels**: Each user has a private channel (`chat.{userId}`) for receiving messages
4. **Message Flow**:
   - User sends a message via POST request to `/api/message/send`
   - Message is saved to the database
   - `MessageSent` event is broadcast to the receiver's private channel
   - React listens for the event and updates the UI in real-time

## 🛠️ Troubleshooting

### WebSocket Connection Issues

- Ensure Reverb is running: `php artisan reverb:start`
- Check CORS configuration in `config/cors.php`
- Verify environment variables match between frontend and backend

### Authentication Errors

- Ensure Sanctum is properly configured
- Check that the auth token is being sent in headers
- Verify SANCTUM_STATEFUL_DOMAINS includes your frontend URL

### Messages Not Appearing

- Check browser console for Echo connection errors
- Verify the channel name matches between backend and frontend
- Ensure the receiver_id is correct in message requests

## 📄 License

This project is open-source and available under the [MIT License](LICENSE).

## 🤝 Contributing

Contributions, issues, and feature requests are welcome!

## 👤 Author

Your Name - [@yourhandle](https://github.com/yourhandle)

---

⭐ If you found this helpful, please give it a star!
