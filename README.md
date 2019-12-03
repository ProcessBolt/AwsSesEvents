# AwsSesEvents
Object definitions for AWS Simple Email Service events from an SQS queue.

# Introduction
If you use Amazon Simple Email Service (SES) to send and/or receive emails with a configuration set that is set up to send send/delivered/bounced/etc. event notifications to a Simple Queueing Service (SQS) queue with a C# application, these object definitions may be of some use.

Amazon supplies these definitions for S3 within the AWSSDK, but not for SES. Furthermore, we could not find anything publicly available. So we decided it would be good to make our object definitions available.

These object defintions were stitched together from varios sources (see references below).

These objects have been designed to make it easy to deserialize the JSON into usable objects using Newtonsoft's ubiquitous Json.NET or similar.

We claim no ownership of the object definitions or any intellectual property. This code is provided only to be helpful to anyone that may need it. Please contribute if you can.

# To Do
The `complaint` objects have not been tested.

# The SQS Message Format
A notifcation fetched from SQS will be in the format represented by `AwsSqsMessage`.  That contains a field called `Message` that contains another JSON object.

That inner JSON object is represented by `AwsSesEvent`, which then in turn may contain different object types depending on the type of event. The `mail` object, represented by `AwsSesEventMail` is common to all event notifications.

Currently, the following event types are implemented:
1. Send (`AwsSesEventSend`)
2. Delivery (`AwsSesEventDelivery`)
3. Open (`AwsSesEventOpen`)
4. Click (`AwsSesEventClick`)
5. Bounce (`AwsSesEventBounce`)
6. Complaint (`AwsSesEventComplaint`)
7. Rejected (`AwsSesEventRejected`)
8. Rendering Failure (`AwsSesEventFailure`)

# Example
```
using System;
using AwsSesEventObjects;
using Amazon.SQS;
using Amazon.SQS.Model;
using Newtonsoft.Json;

namespace AwsSesEventTester
{
    class Program
    {
        static string AwsSqsQueueUrl = "https://sqs.us-west-2.amazonaws.com";
        static string AwsSqsServiceUrl = "https://sqs.us-west-2.amazonaws.com/1234567890/Some-Queue-Name";

        static void Main(string[] args)
        {
            AmazonSQSConfig sqsConfig;
            AmazonSQSClient sqsClient;
            ReceiveMessageResponse receiveMessageResponse;

            sqsConfig = new AmazonSQSConfig
            {
                ServiceURL = AwsSqsServiceUrl
            };

            sqsClient = new AmazonSQSClient(sqsConfig);

            ReceiveMessageRequest receiveMessageRequest = new ReceiveMessageRequest
            {
                QueueUrl = AwsSqsQueueUrl
            };

            receiveMessageResponse = sqsClient.ReceiveMessage(receiveMessageRequest);

            foreach (Message message in receiveMessageResponse.Messages)
            {
                ProcessMessageEvent(sqsClient, message);
            }
        }

        private static void ProcessMessageEvent(AmazonSQSClient sqsClient, Message message)
        {
            AwsSqsMessage sqsMessage = JsonConvert.DeserializeObject<AwsSqsMessage>(message.Body);

            if (!sqsMessage.Message.StartsWith("{\"eventType\":"))
            {
                Console.WriteLine("Not an SES event entry: {0}", sqsMessage.Message);
            }
            else
            {
                AwsSesEvent sesEvent = JsonConvert.DeserializeObject<AwsSesEvent>(sqsMessage.Message);

                switch (sesEvent.EventType)
                {
                    case "Send":
                        Console.WriteLine("Send {0}", sesEvent.Mail.MessageId);
                        break;
                    case "Delivery":
                        Console.WriteLine("Delivery {0} {1}", sesEvent.Mail.MessageId, sesEvent.Delivery.SmtpResponse);
                        break;
                    case "Open":
                        Console.WriteLine("Open {0} {1}", sesEvent.Mail.MessageId, sesEvent.Open.IpAddress);
                        break;
                    case "Click":
                        Console.WriteLine("Click {0} {1}", sesEvent.Mail.MessageId, sesEvent.Click.Link);
                        break;
                    case "Bounce":
                        Console.WriteLine("Bounce {0} {1}", sesEvent.Mail.MessageId, sesEvent.Bounce.BouncedRecipients[0].EmailAddress);
                        break;
                    default:
                        Console.WriteLine("UNIMPLEMENTED SES EVENT TYPE: {0}", sesEvent.EventType);
                        break;
                }
            }

            // Message can be removed from queue now
        }
    }
}
```
# References
The following references (including but not limited to) were used as documentation to create these object definitions:
https://docs.aws.amazon.com/ses/latest/DeveloperGuide/receiving-email-notifications.html
https://docs.aws.amazon.com/ses/latest/DeveloperGuide/event-publishing-retrieving-firehose-contents.html
https://docs.aws.amazon.com/ses/latest/DeveloperGuide/receiving-email-notifications-contents.html
https://aws.amazon.com/blogs/messaging-and-targeting/introducing-sending-metrics/
https://docs.aws.amazon.com/ses/latest/DeveloperGuide/receiving-email.html
