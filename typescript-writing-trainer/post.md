Ever since I started studying philosophy during my sabbatical last year, writing has been one of the skills I wanted to improve on outside of university, since it's a core skill within academia, but also highly useful across just about any other area.

I'm not very confident in my writing skills both in German and English, even though I've been writing for this blog for several years and also did some "professional" writing in the form of writing blog posts for [Logrocket](https://blog.logrocket.com/author/mariozupan/). The reason for this is that after school, I've never actively worked on writing as a skill. I write and I try to incorporate the feedback I get, but essentially I rely on my spoken language skills and a lot of reading to carry me through so far.

However, whenever I read anything by an experienced and skilled writer, I can't help but notice the difference to my own writing. Reading a well written article, book, essay, etc. is a pristine experience and can by itself push the written piece towards its goal.

Luckily, during the course of my philosophy studies, I will go through some formal education on writing, especially concerning academic writing. However, I figure that there are some parallels to learning how to program, which is what I spent most of my time during the last decade in that there's always room to improve and a university course ranging over a semester is just not enough to get good. Getting good at anything takes time, it takes repetition in the form of [deliberate practice](https://www.goodreads.com/book/show/26312997-peak). 

I asked some of my friends who write more and way better than me, what are some good starting points to get better at writing and one suggestion was to do short writing drills. These are small writing exercises aimed at getting more comfortable with writing and to improve vocabulary and style at the same time without any pressure of having to publish the nonsense you're writing afterwards.

The idea is to grab a random topic, or a random list of words and, for a specific amount of time, say 7 minutes, just write and don't stop. That's it. Afterwards you can delete what you wrote, or, if you like what you did, keep it around and maybe continue working on it.

I thought that's a cool idea, as it reminded me of programming katas and of the exercises we did during coding dojos, where we'd come up with a certain restriction ( e.g.: no IF's, no loops) and implement the same simple program (e.g. Conway's Game of Life) with these restrictions as an exercise and afterwards deleted the code.

Cool, so we have an idea for an exercise and of course, being a programmer at heart, the first thing I thought was "that's a great use-case for playing around with TypeScript, which I've been wanting to do for a while" and here we are.

We'll build a simplistic Express backend API using TypeScript, which will just provide us a list of random words requested from our frontend. This will be similar to [this cool project](https://github.com/RazorSh4rk/random-word-api/) where I stole the language files, but as I said, much more simple.

Also, we will build a UI for the app, where we can configure the amount of words we want to use, the length of time we want to write in minutes and a language switcher between German and English. There will also be a text field to write in, a timer and an indicator which of the words we already used in our text.

Awesome, let's go!

## Server Implementation

Let's start with the server, since it's very simple. We'll base the whole thing on [Express](https://expressjs.com/) and we'll just create two endpoints:

* `GET /words/de/$number` - returns $number German words
* `GET /words/en/$number` - returns $number English words

The only dependencies we'll use are `typescript`, `express` and `cors`.

We'll put the above mentioned JSON files containing words in German and English in `./languages/de.json` and `./languages/en.json`.

Let's just look at the whole server code at once at walk through it, as it's quite simple and short.
```typescript
import express, { Application, Router, Request, Response } from 'express';
import cors from 'cors';
import * as fs from 'fs';

const de_words: String[] = JSON.parse(fs.readFileSync('./languages/de.json', {encoding:'utf8', flag:'r'}));
const en_words: String[] = JSON.parse(fs.readFileSync('./languages/en.json', {encoding:'utf8', flag:'r'}));

const app: Application = express();
const route = Router();

route.get("/words/de/:num", async(req: Request, res: Response): Promise<any> => {
    const num: number = parseInt(req.params.num);

    if (num > 100 || num < 1) {
        res.status(400);
        return res.json({
            error: "number needs to be between 1 and 100"
        })
    }

    return res.json({
        words: n_random_words_from_file(de_words, num),
    });
});

route.get("/words/en/:num", async(req: Request, res: Response): Promise<any> => {
    const num: number = parseInt(req.params.num);

    if (num > 100 || num < 1) {
        res.status(400);
        return res.json({
            error: "number needs to be between 1 and 100"
        })
    }

    return res.json({
        words: n_random_words_from_file(en_words, num),
    });
});

function rand(min: number, max: number) {
    return Math.floor(
        Math.random() * (max - min) + min
    )
}

function n_random_words_from_file(file: String[], num: number): String[] {
    const len = file.length;
    const random_nums: number[] = [];
    let i = 0;

    while (i < num) {
        let new_rand = rand(0, len);
        if (random_nums.includes(new_rand)) {
            continue;
        }
        i++;
        random_nums.push(new_rand);
    }
    return random_nums.map((n) => file[n])
}


app.use(express.json());
app.use(cors());
app.use(route);

app.listen(8080, () => {
    console.log("Server running on port 8080");
});
```

First, we use `fs.readFileSync` and `JSON.parse` to read the two language files into memory, before the server starts. Then we configure the Express app and add our two routes `/words/de` and `/words/en` with a numeric parameter `:num`.

Within the handler, we parse the numeric parameter, validate that it's between 1 and 100, returning an error if it isn't,  and return a JSON result of the form:

```json
{
    "words": ["one", "two"]
}
```

We get the data by calling the helper function `n_random_words_from_file`, which we pass the numeric parameter and the JSONified contents of our language file.

Within this function, we first determine the amount of words in the file and simply create unique `n` random numbers, where `n` is the numeric value given by the client, adding these numbers to an array and returning an array of these numbers mapped to the n'th position within the language file, giving us `n` random words within the given language file.

Then we just finish the Express config, adding a default CORS config, setting up our router and start the app on port `8080`. That's it.

For convenience, we also add a `run.sh` script to compile typescript and start the server

```bash
#!/bin/bash -e

./node_modules/.bin/tsc && node build/index.js
```

That's it for the API - as I said, it's very simple and the impact TS had in such a simple app was minimal, so I won't even go into it and instead we'll move on to the UI implementation.

## Client Implementation

```typescript
```

```typescript
```

```typescript
```

```typescript
```

```typescript
```

```typescript
```

That's it - you can find the whole code [here](https://github.com/zupzup/writing-trainer) and this is how it looks:

<center>
    <a href="images/screen.png" target="_blank"><img src="images/screen_thmb.png" /></a>
</center>

## Conclusion

I did some writing sessions with this bad boy and it's actually quite fun. Whether I'm getting better at writing is something we'll hopefully see over the next months and years, but I can definitely see a training exercise like this to be recurring part of a scheduled program focused on deliberate practice concerning writing.

Also, having used TypeScript for the first time in anything I wrote myself (I did edit and review some TS code before from other people), I have to say it definitely seems like the game changer it has been talked about to me by my JavaScript friends and colleagues. Especially coming from Rusts fantastic strict type system and compiler, having this amount of safety in a language like JavaScript is a breath of fresh air when it comes to frontend development. This little project also gave me the opportunity to play around with "modern" (i.e. 2022) React and the new modes for handling state in the form of hooks (I was still used to 2015/2016 React using Redux :D).

All in all, this has been a great experience and I can definitely see myself building some more fun stuff with TypeScript.

#### Resources

* [TypeScript](https://www.typescriptlang.org/)
* [React](https://reactjs.org/)
* [Express](https://expressjs.com/)
