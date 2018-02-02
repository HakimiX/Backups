var express = require('express');
var request = require('request');
var rssReader = require('feed-read');
var timers = require('timers');
var http = require('http');
var router = express.Router();

// User Model
var User = require('../model/user');

var token = "";
var googleNewsEndpoint = "https://news.google.com/news?output=rss";

var wit_endpoint = '';
var wit_token = '';

// BodyObjects
var ArticleBodyObj;
var SpecificPersonBodyObj;

var firstname = "inger";
var lastname = "støjberg";
//var name = "inger%støjberg";

var jokesItem;
var jokesArray = [
  "Hvad er det værste ved politiske vittigheder? At de bliver valgt til Folketinget!",
  "Hvad er Donald Trump's foretrukne nation? Diskrimination!",
  "Det er kun politikere, der kan stå oprejst, uden at have en rygrad!",
  "Hvad er forskellen mellem en flyvende gris og en politiker? Bogstavet F!",
  "Hvis kvinder styrede verden, ville der ikke være nogen krige. Krige kræver nemlig strategi og logik!",
  "Hvorfor kan Donald Trump ikke være en Lannister? Fordi han aldrig betaler sin gæld!"
];

// API - Consume Articles
function getArticles() {
  var options = {
      host: '',
      port: 80,
      path: '/Articles/DetailsTop5',
      method: 'GET'
  };
  
  http.request(options, function(res) {
      var body = '';
  
      res.on('data', function(chunk) {
          body+= chunk;
      });
  
      res.on('end', function() {
          ArticleBodyObj = JSON.parse(body);
          console.log(ArticleBodyObj);
      })
  }).end();
}

// API - Consume Specific Person
function getSpecificPerson(name) {
    var options = {
        host: '',
        port: 80,
        path: '/Person/DetailsByName?name=' + name,
        method: 'GET'
    };
    
    http.request(options, function(res) {
        var body = '';
    
        res.on('data', function(chunk) {
            body+= chunk;
        });
    
        res.on('end', function() {
            SpecificPersonBodyObj = JSON.parse(body);
            console.log(SpecificPersonBodyObj);
        })
    }).end();
}

/* GET home page. */
router.get('/', function(req, res, next) {
  res.render('index', { title: 'Express' });
});

router.get('/webhook/', function(req, res) {
  if (req.query['hub.verify_token'] === 'verify_me') {
    res.send(req.query['hub.challenge']);
  }
  res.send('Error, wrong validation token');
});

router.post('/webhook/', function (req, res) {
  messaging_events = req.body.entry[0].messaging;
  for(i = 0; i < messaging_events.length; i++) {
    event = req.body.entry[0].messaging[i];
    sender = event.sender.id;

    if(event.message && event.message.text) {
      text = event.message.text;

      substring = "hvem er";

      if(text.indexOf(substring) !== -1) {

          var length = text.length;
          var trimmedString = text.substring(8, length);
          var normalize = trimmedString.toLowerCase().replace(/ /g, '%');

          getSpecificPerson(normalize);
          timers.setTimeout(() => sendSpecificPersonMessage(sender), 2000);
          
      } else {

        switch(text) {
            case "bum":
                sendText(sender, "switch 2");
                break;
            case "list": 
              sendListTemplate(sender);
              break;
            case "video":
              sendVideo(sender);
              sendVideo2(sender);
              break;
            case "videos":
              sendVideos(sender);
              break;
            case "play":
              sendPlay(sender);
              break;
            case "poul hansen":
              getSpecificPerson();
              timers.setTimeout(() => sendSpecificPersonMessage(sender), 2000);
              break;
            default:
              callWithAI(text, function(err, intent) {
                handleIntent(intent, sender);
              })
          }
      }

      //sendText(sender, text);
    }
    if (event.postback) {
      let text = JSON.stringify(event.postback)
      
      switch(text) {
        case "test":
          sendText(sender, "test postback");
          break;
        default: 
          callWithAI(text, function(err, intent) {
            handleIntent(intent, sender);
          })
      }

    }
  }
  res.sendStatus(200);
});


// Wit AI Connection
function callWithAI(query, callback) {
  query = encodeURIComponent(query);
  request({
    uri: wit_endpoint+query,
    qs: {access_token: wit_token},
    method: 'GET'
  }, function(error, response, body) {
    if(!error && response.statusCode == 200) {
      console.log('Successfully got %s', response.body);
      try {
        body = JSON.parse(response.body)
        intent = body['entities']['intent'][0]['value']
        callback(null, intent)
      } catch (e) {
        callback(e)
      }
    } else {
      console.log(response.statusCode)
      console.error("Unable to send message. %s", error);
      callback(error)
    }
  });
}

// Wit AI Intents
function handleIntent(intent, sender) {
  switch(intent) {
    case "greeting":
      sendText(sender, "Hej, hvad kan jeg hjælpe dig med?");
      break;
    case "jokes":
      sendText(sender, jokesItem = jokesArray[Math.floor(Math.random()*jokesArray.length)].toString());
      break;
    case "laugh":
      sendText(sender, "hehe ;)");
      break;
    case "identification":
      sendText(sender, "Jeg er Altingets Chatbot");
      break;
    case "identification_name":
      sendText(sender, "Mit navn er Altbot ;)");
      break;
    case "help":
      sendHelp(sender);
      break;
    case "artikler":
      getArticles();  
      timers.setTimeout(() => sendText(sender, "Seneste nyheder"), 2000);
      //sendText(sender, "Okay, her er hvad jeg fandt");
      timers.setTimeout(() => sendArticleMessage(sender), 2500);
      break;
    case "finish":
      sendText(sender, "Det var så lidt :)");
      break;
    default:
      sendTextMessage(sender, "Jeg forstår desværre ikke hvad du mener :(")
      break;
  }
}


function sendTextMessage(sender, text) {
  let messageData = {text: text}
  request({
      url: 'https://graph.facebook.com/v2.11/me/messages',
      qs: {access_token: token},
      method: 'POST',
      json: {
          recipient: {id: sender},
          message: messageData 
      }
  }, function(error, response, body) {
      if(error) {
          console.log('sending error')
      } else if(response.body.error) {
          console.log('response body error')
      }
  })
}

function sendText(sender, text) {
  let messageData = {text: text}
  request({
      url: 'https://graph.facebook.com/v2.11/me/messages',
      qs: {access_token: token},
      method: 'POST',
      json: {
          recipient: {id: sender},
          message: messageData 
      }
  }, function(error, response, body) {
      if(error) {
          console.log('sending error')
      } else if(response.body.error) {
          console.log('response body error')
      }
  })
}

function sendArticleMessage(sender) {
  let messageData = {
      "attachment": {
          "type": "template",
          "payload": {
              "template_type": "generic",
              "elements": [{
                  "title": ArticleBodyObj[17].Headline,
                  "subtitle": ArticleBodyObj[17].CreateTime,
                  "image_url": "https://www.altinget.dk/images/article/149600/25205.jpg",
                  "buttons": [{
                      "type": "web_url",
                      "url": "https://www.altinget.dk/artikel/" + ArticleBodyObj[17].UrlKey,
                      "title": "Læs mere"
                  }, {
                      "type": "postback",
                      "title": "Postback",
                      "payload": "Payload for first element in a generic bubble",
                  }],
              }, {
                  "title": ArticleBodyObj[18].Headline,
                  "subtitle": ArticleBodyObj[18].CreateTime,
                  "image_url": "http://ziremedia.com/wp-content/uploads/2017/04/mobile-optimized-1030x686.jpg",
                  "buttons": [{
                      "type": "web_url",
                      "url": "https://www.altinget.dk/artikel/" + ArticleBodyObj[18].UrlKey,
                      "title": "Læs mere",
                  }, {
                      "type": "postback",
                      "title": "Postback",
                      "payload": "Payload for second element in a generic bubble",
                  }],
              }, {
                  "title": ArticleBodyObj[19].Headline,
                  "subtitle": ArticleBodyObj[19].CreateTime,
                  "image_url": "https://www.altinget.dk/images/article/149598/25203.jpg",
                  "buttons": [{
                      "type": "web_url",
                      "url": "https://www.altinget.dk/artikel/" + ArticleBodyObj[19].UrlKey,
                      "title": "Læs mere",
                  }, {
                      "type": "postback",
                      "title": "Postback",
                      "payload": "Payload for second element in a generic bubble",
                  }],
              }, {
                  "title": ArticleBodyObj[20].Headline,
                  "subtitle": ArticleBodyObj[20].CreateTime,
                  "image_url": "https://www.altinget.dk/images/article/149597/25202.jpg",
                  "buttons": [{
                      "type": "web_url",
                      "url": "https://www.altinget.dk/artikel/" + ArticleBodyObj[20].UrlKey,
                      "title": "Læs mere",
                  }, {
                      "type": "postback",
                      "title": "Postback",
                      "payload": "Payload for second element in a generic bubble",
                  }],
              }, {
                  "title": ArticleBodyObj[21].Headline,
                  "subtitle": ArticleBodyObj[21].CreateTime,
                  "image_url": "https://www.altinget.dk/images/article/149596/25201.jpg",
                  "buttons": [{
                      "type": "web_url",
                      "url": "https://www.altinget.dk/artikel/" + ArticleBodyObj[21].UrlKey,
                      "title": "Læs mere",
                  }, {
                      "type": "postback",
                      "title": "Postback",
                      "payload": "Payload for second element in a generic bubble",
                  }],
              }]
          }
      }
  }
  request({
      url: 'https://graph.facebook.com/v2.11/me/messages',
      qs: {access_token:token},
      method: 'POST',
      json: {
          recipient: {id:sender},
          message: messageData,
      }
  }, function(error, response, body) {
      if (error) {
          console.log('Error sending messages: ', error)
      } else if (response.body.error) {
          console.log('Error: ', response.body.error)
      }
  })
}

function sendPersonsMessage(sender) {
  let messageData = {
      "attachment": {
          "type": "template",
          "payload": {
              "template_type": "generic",
              "elements": [{
                  "title": "Inger Støjberg, Politiker",
                  "subtitle": "Element #1 of an hscroll",
                  "image_url": "http://www.venstre.dk/_Resources/Persistent/50355aeccd8d943149f213601b9d80dab6efbd0d/Inger%20St%C3%B8jberg-2202x2544-866x1000.jpg",
                  "buttons": [{
                      "type": "web_url",
                      "url": "https://www.altinget.dk/",
                      "title": "web url"
                  }, {
                      "type": "postback",
                      "title": "Postback",
                      "payload": "Payload for first element in a generic bubble",
                  }],
              }, {
                  "title": "Johanne Schmidt Nielsen, Politiker",
                  "subtitle": "Element #2 of an hscroll",
                  "image_url": "http://denoffentlige.dk/sites/default/files/johanne_schmidt-nielsen_foto_mark_knudsen.jpg",
                  "buttons": [{
                      "type": "web_url",
                      "url": "https://www.altinget.dk/",
                      "title": "web url",
                  }, {
                      "type": "postback",
                      "title": "Postback",
                      "payload": "Payload for second element in a generic bubble",
                  }],
              }]
          }
      }
  }
  request({
      url: 'https://graph.facebook.com/v2.11/me/messages',
      qs: {access_token:token},
      method: 'POST',
      json: {
          recipient: {id:sender},
          message: messageData,
      }
  }, function(error, response, body) {
      if (error) {
          console.log('Error sending messages: ', error)
      } else if (response.body.error) {
          console.log('Error: ', response.body.error)
      }
  })
}

function sendSpecificPersonMessage(sender) {
    let messageData = {
        "attachment": {
            "type": "template",
            "payload": {
                "template_type": "generic",
                "elements": [{
                    "title": SpecificPersonBodyObj[0].PersonName,
                    "subtitle": SpecificPersonBodyObj[0].PersonTitle,
                    "image_url": "https://www.altinget.dk/images/person/" + SpecificPersonBodyObj[0].PersonRecordId + "/" + SpecificPersonBodyObj[0].PersonImage,
                    "buttons": [{
                        "type": "web_url",
                        "url": "https://www.altinget.dk/person/" + SpecificPersonBodyObj[0].PersonUrlKey,
                        "title": "Læs mere"
                    }, {
                        "type": "postback",
                        "title": "Postback",
                        "payload": "Payload for first element in a generic bubble",
                    }],
                }]
            }
        }
    }
    request({
        url: 'https://graph.facebook.com/v2.11/me/messages',
        qs: {access_token:token},
        method: 'POST',
        json: {
            recipient: {id:sender},
            message: messageData,
        }
    }, function(error, response, body) {
        if (error) {
            console.log('Error sending messages: ', error)
        } else if (response.body.error) {
            console.log('Error: ', response.body.error)
        }
    })
}


function sendHelp(sender) {
  let messageData = {
    "attachment": {
        "type": "template",
        "payload": {
            "template_type": "button",
            "text": "Jeg kan vise dig seneste nyheder fra Altinget. Du skal blot skrive at du vil have nyhederne eller trykke på nedenstående knap",
            "buttons": [{
                "type": "postback",
                "title": "Nyheder",
                "payload": "Payload for first element in generic bubble",
            }]
        }
    }
}
request({
    url: 'https://graph.facebook.com/v2.11/me/messages',
    qs: {access_token:token},
    method: 'POST',
    json: {
        recipient: {id:sender},
        message: messageData,
    }
}, function(error, response, body) {
        if (error) {
            console.log('Error sending messages: ', error)
        } else if (response.body.error) {
            console.log('Error: ', response.body.error)
        }
    })
}


function sendListTemplate(sender) {
    let messageData = {
        "attachment": {
            "type": "template",
            "payload": {
              "template_type": "list",
              "top_element_style": "large",
              "elements": [
                {
                  "title": "Classic T-Shirt Collection",
                  "subtitle": "See all our colors",
                  "image_url": "https://www.altinget.dk/images/article/163878/23075.jpg",
                  "buttons": [
                    {
                      "title": "Læs mere",
                      "type": "web_url",
                      "url": "https://www.altinget.dk/artikel/socialdemokratisk-veteran-stopper",
                      "messenger_extensions": true,
                      "webview_height_ratio": "full",
                      "fallback_url": "https://www.altinget.dk"            
                    }
                  ]
                },
                {
                  "title": "Classic White T-Shirt",
                  "subtitle": "See all our colors",
                  "image_url": "https://www.altinget.dk/images/article/163878/23075.jpg",
                  "default_action": {
                    "type": "web_url",
                    "url": "https://www.altinget.dk",
                    "messenger_extensions": false,
                    "webview_height_ratio": "tall"
                  }
                },
                {
                  "title": "Classic Blue T-Shirt",
                  "image_url": "https://www.altinget.dk/images/article/163847/37543.jpg",
                  "subtitle": "100% Cotton, 200% Comfortable",
                  "default_action": {
                    "type": "web_url",
                    "url": "https://www.altinget.dk/artikel/michael-kristiansen-naar-politik-er-vaerst-kan-vi-haeve-standarden",
                    "messenger_extensions": true,
                    "webview_height_ratio": "tall",
                    "fallback_url": "https://www.altinget.dk"
                  },
                  "buttons": [
                    {
                      "title": "Læs mere",
                      "type": "web_url",
                      "url": "https://www.altinget.dk/images/article/163714/37418.jpg",
                      "messenger_extensions": true,
                      "webview_height_ratio": "tall",
                      "fallback_url": "https://www.altinget.dk"            
                    }
                  ]        
                }
              ],
               "buttons": [
                {
                  "title": "View More",
                  "type": "postback",
                  "payload": "payload"            
                }
              ]  
            }
          } 
    }
    request({
        url: 'https://graph.facebook.com/v2.11/me/messages',
        qs: {access_token:token},
        method: 'POST',
        json: {
            recipient: {id:sender},
            message: messageData,
        }
    }, function(error, response, body) {
            if (error) {
                console.log('Error sending messages: ', error)
            } else if (response.body.error) {
                console.log('Error: ', response.body.error)
            }
        })
}


function sendPlay(sender){
    let messageData = {
        "attachment":{
            "type":"template",
            "payload":{
              "template_type":"open_graph",
              "elements":[
                 {
                  "url":"https://open.spotify.com/track/7GhIk7Il098yCjg4BQjzvb",
                  "buttons":[
                    {
                      "type":"web_url",
                      "url":"https://en.wikipedia.org/wiki/Rickrolling",
                      "title":"View More"
                    }]      
                }]
            }
        }
    }
    request({
        url: 'https://graph.facebook.com/v2.11/me/messages',
        qs: {access_token:token},
        method: 'POST',
        json: {
            recipient: {id:sender},
            message: messageData,
        }
    }, function(error, response, body) {
            if (error) {
                console.log('Error sending messages: ', error)
            } else if (response.body.error) {
                console.log('Error: ', response.body.error)
            }
        })
}


function sendVideo(sender) {
    let messageData = {
        "attachment": {
            "type": "template",
            "payload": {
                "template_type": "media",
                "elements": [{
                    "media_type": "video",
                    "url": "https://www.facebook.com/Altingetdk/videos/10156010402693187/"
                }]
            }
        }
    }
    request({
        url: 'https://graph.facebook.com/v2.11/me/messages',
        qs: {access_token:token},
        method: 'POST',
        json: {
            recipient: {id:sender},
            message: messageData,
        }
    }, function(error, response, body) {
            if (error) {
                console.log('Error sending messages: ', error)
            } else if (response.body.error) {
                console.log('Error: ', response.body.error)
            }
        })
}


function sendVideo2(sender) {
    let messageData = {
        "attachment": {
            "type": "template",
            "payload": {
                "template_type": "media",
                "elements": [{
                    "media_type": "video",
                    "url": "https://www.facebook.com/Altingetdk/videos/10156012279023187/"
                }]
            }
        }
    }
    request({
        url: 'https://graph.facebook.com/v2.11/me/messages',
        qs: {access_token:token},
        method: 'POST',
        json: {
            recipient: {id:sender},
            message: messageData,
        }
    }, function(error, response, body) {
            if (error) {
                console.log('Error sending messages: ', error)
            } else if (response.body.error) {
                console.log('Error: ', response.body.error)
            }
        })
}


function sendVideos(sender) {
    let messageData = {
        "attachment": {
            "type": "template",
            "payload": {
                "template_type": "media",
                "elements": [{
                    "media_type": "video",
                    "url": "https://www.facebook.com/Altingetdk/videos/10156010402693187/"
                }, {
                    "media_type": "video",
                    "url": "https://www.facebook.com/Altingetdk/videos/10156012279023187/"
                }]
            }
        }
    }
    request({
        url: 'https://graph.facebook.com/v2.11/me/messages',
        qs: {access_token:token},
        method: 'POST',
        json: {
            recipient: {id:sender},
            message: messageData,
        }
    }, function(error, response, body) {
            if (error) {
                console.log('Error sending messages: ', error)
            } else if (response.body.error) {
                console.log('Error: ', response.body.error)
            }
        })
}



module.exports = router;

