# Overview

The Web Push protocol allows your application to easily engage users by sending notifications to the browser. The subscription to these notifications is done by the user (opt-in).

The notification types depend on the application. For example, it could be a notification for an internal message or an alert before account closure.

We will see in this documentation that the Web Push API offers several options to customize the notifications by adding buttons, vibration schema, images, urgency indicator and more.

## How Web Push Works

Web Push involves three main actors:

1. **The User Agent (Browser)**: The user's web browser that receives and displays notifications
2. **The Push Service**: A service provided by the browser vendor (e.g., Google FCM, Mozilla Push Service) that routes notifications
3. **Your Application Server**: Your backend that sends notifications

### The Web Push Flow

```
1. User visits your website
   ↓
2. User grants notification permission
   ↓
3. Browser creates a subscription with the Push Service
   ↓
4. Subscription data is sent to your Application Server
   ↓
5. Your Application Server stores the subscription
   ↓
6. When needed, your Application Server sends a notification
   ↓
7. Push Service delivers to the Browser
   ↓
8. Browser displays the notification to the user
```

### Key Concepts

**Subscription**: A unique endpoint and set of keys that identify a user's device and browser. Each user can have multiple subscriptions (desktop, mobile, etc.).

**VAPID**: Voluntary Application Server Identification authenticates your server to the push service, preventing unauthorized parties from sending notifications.

**Payload Encryption**: All notification content is encrypted end-to-end. Only the user's browser can decrypt the message.

**Service Worker**: A background script that runs independently of your web page and handles push events even when the site isn't open.

## Use Cases

Web Push notifications are ideal for:

- **Real-time Updates**: Chat messages, social media interactions, breaking news
- **Transactional Notifications**: Order confirmations, shipping updates, payment receipts
- **Engagement**: Abandoned cart reminders, content recommendations, event reminders
- **Alerts**: System status, security alerts, time-sensitive information
- **Collaboration**: Team notifications, document changes, mentions

## Benefits of Web Push

### For Users
- **Timely Information**: Receive updates instantly without checking the app
- **Control**: Users must explicitly opt-in and can unsubscribe at any time
- **Cross-Device**: Works on desktop and mobile browsers
- **No App Installation**: Works directly in the browser, no app download required

### For Developers
- **Standard Protocol**: Based on open web standards (RFC 8030, 8291, 8292)
- **Cross-Platform**: One implementation works across Chrome, Firefox, Safari, Edge
- **Scalable**: Push services handle the infrastructure
- **Reliable Delivery**: Guaranteed delivery even when the browser is closed

## Browser Support

Web Push is supported by all modern browsers:

- ✅ Chrome (Desktop & Android) - Full support
- ✅ Firefox (Desktop & Android) - Full support
- ✅ Safari (Desktop & iOS 16.4+) - Full support
- ✅ Edge (Desktop & Android) - Full support
- ✅ Opera (Desktop & Android) - Full support

Check the latest browser compatibility at [https://caniuse.com/push-api](https://caniuse.com/push-api)

## Requirements

To implement Web Push notifications, you need:

1. **HTTPS**: Web Push only works on secure origins (https:// or localhost)
2. **Service Worker**: A registered service worker to handle push events
3. **User Permission**: Explicit user consent to receive notifications
4. **VAPID Keys**: Public/private key pair to authenticate your server
5. **Backend Implementation**: Server-side code to send notifications (this library!)

## Security & Privacy

Web Push is designed with security and privacy in mind:

- **End-to-End Encryption**: All notification payloads are encrypted
- **User Consent**: Users must explicitly grant permission
- **Revocable**: Users can revoke permission at any time
- **VAPID Authentication**: Prevents unauthorized sending
- **No Tracking**: Push services don't have access to notification content

## Testing

You want to test it? Please go to [this demo page](https://serviceworke.rs/push-payload_demo.html) to see what your browser already supports.

## Next Steps

Now that you understand the basics, explore the detailed documentation:

- [The Subscription](the-subscription.md) - Learn about subscription objects and content encodings
- [The Notification](the-notification.md) - Discover how to create and customize notifications
- [The Status Report](the-status-report.md) - Understand delivery status and error handling
- [VAPID](vapid.md) - Set up server authentication

Then, choose your implementation:

- [The Library](../the-library/installation.md) - Standalone PHP library
- [The Symfony Bundle](../the-symfony-bundle/installation.md) - Symfony integration
