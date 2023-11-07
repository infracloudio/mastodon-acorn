# Mastodon Server on Acorn
===


Mastodon is a free and open-source software for running self-hosted social networking services. It has microblogging features similar to Twitter, which are offered by a large number of independently run nodes, known as instances or servers, each with its own code of conduct, terms of service, privacy policy, privacy options, and content moderation policies.

You can simply run your own mastodon server.

## Configure Mastodon server.

All the components are added you just need to bring your own smtp server and you are ready to lauch your own mastodon server.
You need to select "Advanaced Options" after giving the image and provide all the details required for smtp server.Once your mastodon server is running you can create the admin user by going to web container and clicking on `Execute shell` this will open up a terminal and you can execute below command which will create the admin user. And it will generate the password for you.


```
RAILS_ENV=production bin/tootctl accounts create \
  alice \
  --email jovon49621@glalen.com \
  --confirmed \
  --role Owner
```
Here `alice` is the name , so change the name and email accordingly.

If you are looking for free tier SMTP server [Brevo](https://www.brevo.com/) is good one.