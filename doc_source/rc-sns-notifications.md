# Configuring Amazon SNS Notifications for Run Command<a name="rc-sns-notifications"></a>

You can configure Amazon Simple Notification Service \(Amazon SNS\) to send notifications about the status of commands you send using Systems Manager Run Command\. Amazon SNS coordinates and manages the delivery or sending of notifications to subscribing clients or endpoints\. You can receive a notification whenever a command changes to a new state or changes to a specific state, such as failed or timed out\. In cases where you send a command to multiple instances, you can receive a notification for each copy of the command sent to a specific instance\. Each copy is called an *invocation*\.

Amazon SNS can deliver notifications as HTTP or HTTPS POST, email \(SMTP, either plain\-text or in JSON format\), or as a message posted to an Amazon Simple Queue Service \(Amazon SQS\) queue\. For more information, see [What Is Amazon SNS](http://docs.aws.amazon.com/sns/latest/dg/) in the *Amazon Simple Notification Service Developer Guide*\.

## Configure Amazon SNS Notifications for Systems Manager<a name="rc-sns"></a>

Run Command supports sending Amazon SNS notifications for commands that enter the following statuses\. For information about the conditions that cause a command to enter one of these statuses, see [Understanding Command Statuses](monitor-commands.md)\.
+ In Progress
+ Success
+ Failed
+ Timed Out
+ Canceled

**Note**  
Commands sent using Run Command also report Canceling and Pending status\. These statuses are not captured by Amazon SNS notifications\.

If you configure Run Command for Amazon SNS notifications, Amazon SNS sends summary messages that include the following information:


****  

| Field | Type | Description | 
| --- | --- | --- | 
|  EventTime  |  String  |  The time the event was triggered\. The time stamp is important because Amazon SNS does not guarantee message delivery order\. Example: 2016\-04\-26T13:15:30Z   | 
|  DocumentName  |  String  |  The name of the SSM document used to run this command\.  | 
|  CommandId  |  String  |  The ID generated by Run Command after the command was sent\.  | 
| ExpiresAfter | Date | If this time is reached and the command has not already started executing, it will not run\.  | 
| OutputS3BucketName | String | The Amazon Simple Storage Service \(Amazon S3\) bucket where the responses to the command execution should be stored\.  | 
| OutputS3KeyPrefix | String | The Amazon S3 directory path inside the bucket where the responses to the command execution should be stored\.  | 
|  RequestedDateTime  |  String  |  The time and date the request was sent to this specific instance\.  | 
|  InstanceId  |  String  |  The instance targeted by the command\.  | 
|  Status  |  String  |  Command status for the command\.  | 

If you send a command to multiple instances, Amazon SNS can send messages about each copy or invocation of the command that include the following information:


****  

| Field | Type | Description | 
| --- | --- | --- | 
|  EventTime  |  String  |  The time the event was triggered\. The time stamp is important because Amazon SNS does not guarantee message delivery order\. Example: 2016\-04\-26T13:15:30Z   | 
|  DocumentName  |  String  |  The name of the Systems Manager document used to run this command\.  | 
|  RequestedDateTime  |  String  |  The time and date the request was sent to this specific instance\.  | 
|  CommandId  |  String  |  The ID generated by Run Command after the command was sent\.  | 
|  InstanceId  |  String  |  The instance targeted by the command\.  | 
|  Status  |  String  |  Command status for this invocation\.  | 

To set up Amazon SNS notifications when a command changes status, you must complete the following tasks\.

**Topics**
+ [Task 1: Create an IAM Role for Amazon SNS Notifications](#rc-iam-notifications)
+ [Task 2: Attach the iam:PassRole Policy to Your Amazon SNS Role](#rc-sns-passpolicy)
+ [Task 3: Create and Subscribe to an Amazon SNS](#rc-configure-sns)
+ [Task 4: Send a Command that Returns Status Notifications](#rc-send-notifications)

### Task 1: Create an IAM Role for Amazon SNS Notifications<a name="rc-iam-notifications"></a>

Use the following procedure to create an IAM role for Amazon SNS notifications\. This service role is used by Systems Manager to trigger Amazon SNS notifications\. 

**To create an IAM service role for Amazon SNS notifications**

1. Open the IAM console at [https://console\.aws\.amazon\.com/iam/](https://console.aws.amazon.com/iam/)\.

1. In the navigation pane, choose **Roles**, and then choose **Create role**\.

1. On the **Select type of trusted entity** page, under **AWS Service**, choose **EC2**\.

1. In the **Select your use case** section, choose **EC2**, and then choose **Next: Permissions**\.

1. On the **Attached permissions policy** page, search for the **AmazonSNSFullAccess** policy, choose it, and then choose **Next: Review**\. 

1. On the **Review** page, type a name in the **Role name** box, and then type a description\.

1. Choose **Create role**\. The system returns you to the **Roles** page\.

1. On the **Roles** page, choose the role you just created to open the **Summary** page\.

1. Choose the **Trust Relationships** tab, and then choose **Edit Trust Relationship**\.

1. Add "ssm\.amazonaws\.com" to the existing policy as the following code snippet illustrates:

   ```
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "",
         "Effect": "Allow",
         "Principal": {
           "Service": "ec2.amazonaws.com",
           "Service": "ssm.amazonaws.com"
         },
         "Action": "sts:AssumeRole"
       }
     ]
   }
   ```
**Note**  
You must add a comma after the existing entry\. "Service": "ec2\.amazonaws\.com"****, or the JSON will not validate\.

1. Choose **Update Trust Policy**\.

1. Copy or make a note of the **Role ARN**\. You will specify this ARN when you send a command that is configured to return notifications\.

1. Leave the **Summary** page open\.

### Task 2: Attach the iam:PassRole Policy to Your Amazon SNS Role<a name="rc-sns-passpolicy"></a>

To receive notifications from the Amazon SNS service, you must attach the iam:PassRole policy to the service role you created in Task 1\.

**To attach the iam:PassRole policy to your Amazon SNS role**

1. In the **Summary** page for the role you just created, choose the **Permissions** tab\.

1. Choose **Add inline policy**\.

1. On the **Create policy** page, choose the **Visual editor** tab\.

1. Choose **Service**, and then choose **IAM**\.

1. Choose **Select actions**\.

1. In the **Filter actions** text box, type PassRole, and then choose the **PassRole** option\.

1. Choose **Resources**\. Verify that **Specific** is selected, and then choose **Add ARN**\.

1. In the **Specify ARN for role** field, paste the Amazon SNS role ARN that you copied at the end of Task 1\. The system autopopulates the **Account** and **Role name with path** fields\.

1. Choose **Add**\.

1. Choose **Review policy**\.

1. On the **Review Policy** page, type a name and then choose **Create Policy**\.

### Task 3: Create and Subscribe to an Amazon SNS<a name="rc-configure-sns"></a>

An Amazon SNS *topic* is a communication channel that Run Command uses to send notifications about the status of your commands\. Amazon SNS supports different communication protocols, including HTTP/S, email, and other AWS services like Amazon SQS\. For the purpose of getting started quickly, we recommend that you start with the email protocol\. For information about how to create a topic, see [Create a Topic](http://docs.aws.amazon.com/sns/latest/dg/CreateTopic.html) in the *Amazon Simple Notification Service Developer Guide*\.

**Note**  
After you create the topic, copy or make a note of the **Topic ARN**\. You will specify this ARN when you send a command that is configured to return status notifications\.

After you create the topic, subscribe to it by specifying an **Endpoint**\. If you chose the Email protocol, the endpoint is the email address where you want to receive notifications\. For more information about how to subscribe to a topic, see [Subscribe to a Topic](http://docs.aws.amazon.com/sns/latest/dg/SubscribeTopic.html) in the *Amazon Simple Notification Service Developer Guide*\.

Amazon SNS sends a confirmation email from *AWS Notifications* to the email address that you specify\. Open the email and choose the link to **Confirm subscription** link\.

You will receive an acknowledgement message from AWS\. Amazon SNS is now configured to receive notifications and send the notification as an email to the email address that you specified\.

### Task 4: Send a Command that Returns Status Notifications<a name="rc-send-notifications"></a>

This section shows you how to send a command that is configured to return status notifications using either the console or the AWS Command Line Interface \(AWS CLI\)\. 

**To send a command from the console that returns notifications**

1. Open the AWS Systems Manager console at [https://console\.aws\.amazon\.com/systems\-manager/](https://console.aws.amazon.com/systems-manager/)\.

1. In the navigation pane, choose **Run Command**\.

   \-or\-

   If the AWS Systems Manager home page opens first, choose the menu icon \(![\[Image NOT FOUND\]](http://docs.aws.amazon.com/systems-manager/latest/userguide/images/menu-icon-small.png)\) to open the navigation pane, and then choose **Run Command**\.

1. Choose **Run command**\.

1. In the **Command document** list, choose a Systems Manager document\.

1. In the **Targets** section, identify the instances where you want to run this operation by specifying tags or selecting instances manually\.
**Note**  
If you choose to select instances manually, and an instance you expect to see is not included in the list, see [Where Are My Instances?](troubleshooting-remote-commands.md#where-are-instances) for troubleshooting tips\.

1. In the **Command parameters** section, specify values for required parameters\.

1. In **Other parameters**:
   + In the **Comment** box, type information about this command\.
   + In **Timeout \(seconds\)**, specify the number of seconds for the system to wait before failing the overall command execution\. 

1. \(Optional\) In **Rate control**:
   + In **Concurrency**, specify either a number or a percentage of instances on which to run the command at the same time\.
**Note**  
If you selected targets by choosing Amazon EC2 tags, and you are not certain how many instances use the selected tags, then limit the number of instances that can run the document at the same time by specifying a percentage\.
   + In **Error threshold**, specify when to stop running the command on other instances after it fails on either a number or a percentage of instances\. For example, if you specify 3 errors, then Systems Manager stops sending the command when the 4th error is received\. Instances still processing the command might also send errors\.

1. In the **Output options** section, if you want to save the command output to a file, select the **Write command output to an Amazon S3 bucket**\. Type the bucket and prefix \(folder\) names in the boxes\.
**Note**  
The S3 permissions that grant the ability to write the data to an S3 bucket are those of the instance profile assigned to the instance, not those of the IAM user performing this task\. For more information, see [Configuring Access to Systems Manager](systems-manager-access.md)\. 

1. In the **SNS Notifications** section, choose **Enable SNS notifications**\.

1. In the **IAM role** field, type or paste the IAM role ARN you created earlier\.

1. In the **SNS topic** field, type or paste the Amazon SNS ARN you created earlier\.

1. In the **Notify me on** field, choose the events for which you want to receive notifications\.

1. In the **Notify me for** field, choose to receive notifications for each copy of a command sent to multiple instances \(invocations\) or the command summary\.

1. Choose **Run**\.

1. Check your email for a message from Amazon SNS and open the email\. Amazon SNS can take a few minutes to send the mail\.

**To send a command that is configured for notifications from the AWS CLI**

1. Open the AWS CLI\.

1. Specify parameters in the following command\.

   ```
   aws ssm send-command --instance-ids "ID-1, ID-2" --document-name "name" --parameters commands=date --service-role ServiceRole ARN --notification-config NotificationArn=SNS ARN
   ```

   For example

   ```
   aws ssm send-command --instance-ids "i-12345678, i-34567890" --document-name "AWS-RunPowerShellScript" --parameters commands=date --service-role arn:aws-cn:iam:: 123456789012:myrole --notification-config NotificationArn=arn:aws-cn:sns:cn-north-1:123456789012:test
   ```

1. Press Enter\.

1. Check your email for a message from Amazon SNS and open the email\. Amazon SNS can take a few minutes to send the mail\.

For more information about configuring Run Command from the command line, see [Amazon EC2 Systems Manager API Reference](http://docs.aws.amazon.com/ssm/latest/APIReference/) and the [Systems Manager AWS CLI Reference](http://docs.aws.amazon.com/cli/latest/reference/ssm/index.html)\.