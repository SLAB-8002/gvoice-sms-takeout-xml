# gvoice-sms-takeout-xml
Convert Google Voice SMS data from Takeout to .xml suitable for use with SMS Backup and Restore.
Input data is a folder of .html, image, and vCard files from Google Takeout.

Working as of September 21, 2024.

This is forked from [SLAB-8002](https://github.com/SLAB-8002), mostly to optionally auto-remove conversations that weren't convertable and were mostly spam, automated messages, etc. 

## Improvements from SLAB-8002
* Offers to auto-delete conversations that will cause issues and are generally not wanted, e.g. converations without attached phone numbers.

After removing those files my conversion of 145,201 messages, 5061 images, and 14 contact cards succeeded in 6 minutes, 30 seconds.

## Improvements from codecivet fork
* Removes the "MMS Sent" and "MMS Received" messages mentioned by codecivet.
* I ran into an issue where image files were not associated with the correct "img src" element in the HTML files, causing about 1/3 of my images to not get imported. The script now builds lists of all "img src" elements found, then normalizes the image filenames contained in the folder and assigns them a custom sorting order so that they will associate to the correct "img src" from the HTML message.
  * This consumed the vast majority of my time working on this script.
  * Google does not make this easy - instead of just putting the actual filename in the "img src" they somehow screwed it up so that the images don't load even if you open the HTML file in a web browser.
  * What I found is Google named the files by using the "img src", but they truncate the "img src" at 50 characters when creating the filename. They also append a parenthesized number if there are duplicate filenames, so this script accounts for all of that.
  * TBH I may have created this issue by neglecting the optional steps below to export and delete all Google Contacts. When you don't follow those steps, Google names all files using the contact name, and some of my contact names are what pushed the "img src" length over 50 characters. The good news is I confirmed that the original script worked really well for finding the correct phone numbers even when Google uses the contact name.
* Had some issues where certain special characters such as apostrophes, quotation marks, and such were not getting handled properly which was causing SMS Backup & Restore to truncate the message text. Added some additional processing to the message content to properly handle those characters.
* Added support for processing vCards in .vcf format. I mostly just took the image processing that it was doing before and updated it to do the same thing for .vcf files.
  * I had a few .vcf files that were just map pins that someone sent me. Not sure why they came across as contact cards, but I didn't want them to stay that way, so I added a section to find those vCards and turn them into a plain text MMS file containing the URL for the map pin. I added another section that does not perform this conversion, then commented it out. If you don't want to convert these, or you don't have any, you can simply comment out my section that performs the conversion and uncomment the other section.
* I found an error with the earlier script where the user's phone number (i.e. my phone number) was not getting added to the <addr> tag for MMS messages. This was causing some of my MMS messages to not appear in the correct conversation, so I added code to find the owner's phone number, pass that to the write_sms_messages and write_mms_messages functions, and append it to the participants list before creating the <addr> tag. The fun part is you can't append it to the participants list until after it creates the "addresses" element inside the \<mms\> tag because the owner's phone number is *not* supposed to go in that element.
* Of course, fixing the above error created another error where it was not recognizing when MMS messages were sent by me and assiging the correct value to the "msg_box" element, so I updated the sent_by_me variable to check whether sender is equal to own_number. As far as I've been able to find, this has fixed the issue.
* Tested on 5312 messages, 145 images, and 23 vCards. Everything imported or converted successfully as far as I can tell.
* Latest commit substantially reduces the processing time. Previous version took around 15-16 minutes; this version processes the above data set in about 12-14 seconds.

## Improvements from brismuth fork
* Support for images (jpg and gif tested, should work with png)
* Timestamps were off for me. This is fixed.
* Added `requirements.txt` so that dependency versions are known good.
* Text formatting may be improved. For example, there is support for `<br>` tags now. It's possible there are also some text output regressions from my changes.
* The header creation uses much less memory. My environment choked on header creation with a 1.5GB output file without my improvements.
* Should work pretty reliably without the (Optional) steps below. I put in a bunch of stuff to work around issues with missing phone numbers.
* Tested on my 1.5GB archive of 75000 messages. There are a lot of corner cases handled now. It runs completely autonomously on my archive without any hacking or workarounds.

## Issues

- For dual or multi-SIM users, SMS Backup & Restore does not support setting SIM identity through the "sub_id" value on Android 14. I asked Synctech about this, and they said it is an Android 14 issue that they have not been able to figure out how to work around. Just know that all of your texts will show up as being associated with your primary SIM.
- When testing this, I had some issues with SMS Backup & Restore finding duplicates. I think this most likely occurred because I was manually editing some of the GVoice HTML outputs to create particular corner cases that didn't occur in my actual data set. When I asked, Synctech informed me they were checking for duplicates using the "date" element, followed by "m_id" (message ID), then "tr_id" (transaction ID). Most likely I inadvertantly created a message that had the same "date" value as another. It only occurred once in my actual data set, and it was an empty plain text MMS, so I didn't try to correct this issue.
- If you encounter this issue and are losing data that you don't want to, the recommended fix is to create a "tr_id" element for MMS messages and assign a UUID to it. This would probably be pretty easy to implement and would give you a unique "tr_id" for every MMS message.

## How to use:
1. (Optional) Export all Google Contacts
1. (Optional) Delete all Google Contacts (this is causes numbers show up for each thread, otherwise Takeout will sometimes only have names. If you want to skip this step, you can, but some messages won't be linked to the right thread if you do. Note that this may remove Contact Photos on iOS if you don't pause syncing on your iOS device)
1. Get Google Voice Takeout and download
1. (Optional) Restore contacts to your account
1. Clone this repo to your computer. Downloading sms.py and requirements.txt should also work.
1. Extract Google Voice Takeout and move the folder into the same folder as this script
1. Open terminal
1. Install python
1. Install pip
1. Create virtual environment (`python -m venv .venv`)
1. Activate virtual environment (`.venv\Scripts\activate.bat` or `source .venv/bin/activate`)
1. Install dependencies (`python -m pip install -r requirements.txt`)
1. `python sms.py`


## Testing with an emulator:
**I STRONGLY recommend using an emulator to test the output before importing to your phone**
1. Open your emulator application of choice (I used Android Studio AVD).
1. Install SMS Backup & Restore.
  * I did this by downloading the APK from [SyncTech](https://www.synctech.com.au/sms-backup-restore/) then installing using ADB: i.e. ` adb install .\SMSBackupRestore-freeProductionRelease-10.20.002.apk`.
  * Alternatively you can use an emulator system image that includes Google Play Store, connect to your Googe account, and install SMS Backup & Restore from the Play Store.
1. Push the file `gvoice-all.xml` to your emulator using ADB, i.e. `adb push .\gvoice-all.xml /sdcard/Documents`.
1. Open SMS Backup & Restore on the emulator.
1. Use the Restore option to import the `gvoice-all.xml` file. When prompted, set SMS Backup & Restore as the default app for SMS messages.
1. Open your messagine app and set as default when prompted.
  ** I recommend using the messaging app without an account for this. For instance, Google Messages asks me if I want to connect it to my Google account or use it without an account. I selected "Use Messages without an account" so that it would not sync the imported messages to my Google account from the emulator.
1. Inspect your output to make sure if appears how you want it. I recommend searching for images and contact cards to make sure those all imported in the output file. Unfortunately the emulator may not catch all errors with MMS messages because it doesn't have the same phone number as your actual device.

* If you need to tweak the script or output and try again, you can either close the emulator and wipe data in Android Studio, then repeat the steps above; or you can delete all messages, either by manually selecting them in your messaging app or by using the Tools option in SMS Backup & Restore to delete them.

## Importing to your phone
1. Copy the file gvoice-all.xml to your phone, then restore from it using SMS Backup and Restore
1. **IMPORTANT** If you don't see the imported messages in your messaging app, go into the application settings and clear data, then restart it. SMS Backup & Restore has a couple of places where it tells you to do this, but I screwed it up and imported mine multiple times on my first attempt, which created lots of duplicate messages.
