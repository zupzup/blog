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

The UI implementation is based on [Create React App](https://create-react-app.dev/docs/adding-typescript/). You can create a new fully wired up React project with TypeScript support using this command:

```bash
npx create-react-app ui --template typescript
```

I'm usually not a big fan of these magic scaffolding tools, but in this case, since I wasn't really interested in how to set up a TypeScript build process with React etc., but rather just how to build a modern React app with TypeScript, I thought it's fine to go this route. For any production-grade application, I would always recommend understanding your build-pipeline 100% and to go for a minimal footprint.

This is a bit more work in the beginning, but understanding how this works will pay dividends down the line and you'll be able to re-use much of what you research, configure and set up once. Anyway, you do you and this is the reason I went this route in this toy project.

We'll start writing the UI from the top down at `App.tsx`, building the state handling mechanism and the basic structure and then implementing the smaller components we'll need for the actual user interface. The CSS and base HTML won't be covered in this post, as it's very simplistic and not relevant to the example, but you can always check out the whole code [here](https://github.com/zupzup/writing-trainer).

Let's start with our application state in `App.tsx`:

```typescript
type State = {
  running: boolean,
  currentTimer: number,
  stopTime: Date | null,
  words: WordState[],
};

export type WordState = {
  word: string,
  used: boolean,
};
```

For this app we don't need much state floating around. We need to know, whether the app is running (i.e. the timer is running down), when the timer should stop, as well as the remaining time and a `WordState`, where for each random word we get, we check if it has been used already. That way, we can show the user which words they already used.

Based on this, we can define our actions and the initial app state.

```typescript
type StartPayload = {
  minutes: number,
  words: string[],
};


export type Action = 
  | { type: 'start'; payload: StartPayload }
  | { type: 'reset' }
  | { type: 'updateWords', payload: WordState[] }
  | { type: 'tick' };

const initialState: State = {
  running: false,
  currentTimer: 0,
  words: [],
  stopTime: null,
};
```

There are 4 types of actions:

* `start` - starts the timer with the given words, and minutes
* `reset` - resets everything back to the initialState
* `updateWords` - update the `WordState` of a word while a user types
* `tick` - happens every second, when the timer is running, to count down the remaining time and rerender the UI Timer

Next, we come to our reducer - the heart of our state management, where we define which state changes should happen based on the different actions.

```typescript
const reducer = (state: State, action: Action): State => {
  switch (action.type) {
    case 'start': {
      const stopTime = new Date();
      stopTime.setMinutes(stopTime.getMinutes() + action.payload.minutes);
      return {
        running: true,
        currentTimer: action.payload.minutes * 60,
        words: action.payload.words.map((word: string) => ({ word, used: false })),
        stopTime,
      }
    }
    case 'tick': {
      let newTimer = state.currentTimer - 1;
      return {
        running: newTimer === 0 ? false : state.running,
        currentTimer: newTimer,
        words: state.words,
        stopTime: state.stopTime,
      }
    }
    case 'updateWords': {
      return {
        running: state.running,
        currentTimer: state.currentTimer,
        words: action.payload,
        stopTime: state.stopTime,
      }
    }
    case 'reset': {
      return initialState;
    }
    default:
      return initialState;
  }
};
```

When a `start` action happens, we calculate the `stopDate`, set the remaining time and set the words from the action payload (we'll get to the data fetching later). The `updateWords` action simply overrides the `words` entry in the state with the newly calculated value in the payload and `reset` resets us to the initial state. That's it for state handling.

Based on this state, our top level `App` component is pretty simple.

```typescript
function App() {
  const [state, dispatch] = useReducer(reducer, initialState);

  const startWriting = (minutes: number, words: number, lang: String) => {
    axios.get(`http://localhost:8080/words/${lang}/${words}`)
      .then((response: Response) => {
        const words = response.data.words;
        dispatch({ type: 'start', payload: {
          minutes,
          words
        }});
      })
      .catch((error: AxiosError) => {
        alert(`error while fetching words: ${error}`);
      });
  };

  const updateText = (event: ChangeEvent<HTMLTextAreaElement>) => {
    const text = event.target.value;
    dispatch({
      type: 'updateWords',
      payload: state.words.map((w: WordState) => {
        return {
          word: w.word,
          used: text.includes(w.word),
        };
      })
    });
  };

  const stopWriting = () => {
    dispatch({ type:'reset' });
  };

  return (
    <div className="App">
      <div className="Inner">
        <Header 
          startWriting={startWriting}
          running={state.running}
          stopWriting={stopWriting}
        />
        <Words words={state.words} />
        <Text updateText={updateText} running={state.running} />
        <Timer 
          timer={state.currentTimer}
          running={state.running}
          dispatch={dispatch}
          stopTime={state.stopTime}
        />
      </div>
    </div>
  );
}
```

We make sure to use our defined `reducer`, initializing our state to `initialState`. Then we define the actions for `startWriting`, `stopWriting` and `updateText`. In the `startWriting` action, we make a `GET` call to our API with the given time, word count and language from the UI. We'll pass this function down to the `Header` component, where it will be called with the values given by the user.
If the request works, we dispatch a `start` action with the given minutes and the random words from our API.

For `stopWriting`, we simply trigger our `reset` action, since in that case we want to reset to our initial state and nothing else. The `updateText` action will be passed to the `Text` component and called there `onChange`. It basically updates the `words` state by checking, if the text within the textarea includes the given word for each word.
This could be implemented more efficiently, but we don't expect any text or word sizes here where this would become an issue.

Then, we define the markup for our top level component. Within the outer styling div's, we start with the `Header` component including the form fields for configuring the app, followed by `Words`, which includes the list of words and their status (used/not used). After that, we have the `Text` component, which essentially just contains a text area the user can type in with the given change handler and finally we have the `Timer` component, which shows the user a timer at the bottom counting down from the amount of seconds the user has given themselves to zero.

Let's look at the components in order, starting with the Header.

```typescript
function Header(props: {
  startWriting: (minutes: number, words: number, lang: String) => void,
  stopWriting: () => void,
  running: boolean,
}) {
  const [numberOfWords, setNumberOfWords] = useState(5);
  const [numberOfMinutes, setNumberOfMinutes] = useState(7);
  const [lang, setLang] = useState("de");

  const handleNumberOfWordsInput = (event: ChangeEvent<HTMLInputElement>) => {
    setNumberOfWords(parseInt(event.target.value || "5"));
  };

  const handleNumberOfMinutesInput = (event: ChangeEvent<HTMLInputElement>) => {
    setNumberOfMinutes(parseInt(event.target.value || "7"));
  };

  const handleLangChange = (event: ChangeEvent<HTMLInputElement>) => {
    setLang(event.target.value);
  };

  return (
    <div className="Header">
      <span>
        <input onChange={handleNumberOfWordsInput} type="text" placeholder="Number of Words" />
      </span>
      <span>
        <input onChange={handleNumberOfMinutesInput} type="text" placeholder="Number of Minutes" />
      </span>
      <span>
        <input checked={lang == "de" ? true : false} type="radio" name="lang" value="de" id="de" onChange={handleLangChange} />
        <span>DE</span>
        <input checked={lang == "en" ? true : false} type="radio" name="lang" value="en" id="en" onChange={handleLangChange} />
        <span>EN</span>
      </span>
      <span>
        {!props.running ? <button onClick={() => { props.startWriting(numberOfMinutes, numberOfWords, lang)}}>Start</button> : <button onClick={() => { props.stopWriting() }}>Stop</button>}
      </span>
    </div>
  );
}
```

As mentioned above, this is where the user can configure their writing trainer run. We have text fields for the `Number of Words` and `Number of Minutes`, as well as a radio element for setting the language to EN, or DE. When the user clicks the `Start` button, we trigger the passed in `startWriting` function. The values of the word count, minutes and language are managed within the component using React's `useState` and basic HTML event handlers.
For passing around handlers such as this and working with DOM events, TypeScript was fantastic. One aspect is that even passing handlers between components, if you forget to pass a parameter or use a wrong one (e.g. when changing something), TypeScript will notice it and show an error. Also, auto-completion support is just on a whole different level when using TypeScript, so even in this very small app, I already noticed the massive benefits switching to TypeScript would have in a huge code base with multiple people working on it.

Anyway, when a timer is running, we display a `Start` button, otherwise a `Stop` button and they trigger their respective actions in each case. Let's move on to the `Words` component.

```typescript
function Words(props: { words: WordState[] }) {
  return (
    <div className="Words">
      {
        props.words.map((word: WordState) => {
          return <span key={`${word.word}`} className={ word.used ? "active" : "inactive"}>
            {word.word}
          </span>
        })
      }
    </div>
  );
}
```

This is a simple one. We pass in the `words` from the state and we map them to span elements with different classes (red, or green color) depending on whether they have been used, or not.

The next component is also very simple, it's essentially just a text area in `Text`.

```typescript
function Text(props: {
  updateText: (event: ChangeEvent<HTMLTextAreaElement>) => void,
  running: boolean,
}) {
  return (
    <div className="Text">
      <textarea disabled={!props.running} placeholder="Start Typing..." onChange={props.updateText} />
    </div>
  );
}

```
As mentioned above, the `Text` component consists only of a textarea with a change handler. We also disable the text area if we're not in `running` state, but that's all that's happening here.

Finally, let's look at the `Timer` component.

```typescript
function Timer(props: { timer: number, running: boolean, stopTime: Date | null,  dispatch: Dispatch<Action> }) {
  useEffect(() => {
    let interval: any;
    if (props.running) {
      interval = setInterval(() => {
        props.dispatch({ type: 'tick' })
      }, 1000);
    } else {
      clearInterval(interval);
    }
    return () => clearInterval(interval);
  });

  const now: number = new Date().valueOf();

  let diff = props.stopTime ? (props.stopTime.valueOf() - now) / 1000 : 0;
  if (diff <= 0) {
    diff = 0;
  }

  return (
    <div className="Timer" key={props.timer}>
      {Math.round(diff)}
    </div>
  );
}
```

Here we can see the use of `useEffect`, which is the replacement for React's lifecycle methods and enables us to trigger side-effects during different points in the component's life.

In this case, we set an `interval` for this component, triggering a `tick` action every 1000 ms if we're in `running` state and clear the interval otherwise. This enables us to tick down the seconds in our state and to re-render this component on every tick by passing in the remaining time. We also return the `clearInterval` function from the component, so that the interval is cleared whenever the component is destroyed.

The rest of the component just consists of calculating the remaining time and displaying it to the user. And with that, we created our UI, which looks like this:

<center>
    <a href="images/screen.png" target="_blank"><img src="images/screen_thmb.png" /></a>
</center>

That's it - you can find the whole code [here](https://github.com/zupzup/writing-trainer).

## Conclusion

I did some writing sessions with this bad boy and it's actually quite fun. Whether I'm getting better at writing is something we'll hopefully see over the next months and years, but I can definitely see a training exercise like this to be recurring part of a scheduled program focused on deliberate practice concerning writing.

Also, having used TypeScript for the first time in anything I wrote myself (I did edit and review some TS code before from other people), I have to say it definitely seems like the game changer it has been talked about to me by my JavaScript friends and colleagues. Especially coming from Rusts fantastic strict type system and compiler, having this amount of safety in a language like JavaScript is a breath of fresh air when it comes to frontend development. This little project also gave me the opportunity to play around with "modern" (i.e. 2022) React and the new modes for handling state in the form of hooks (I was still used to 2015/2016 React using Redux :D).

All in all, this has been a great experience and I can definitely see myself building some more fun stuff with TypeScript.

#### Resources

* [TypeScript](https://www.typescriptlang.org/)
* [React](https://reactjs.org/)
* [Express](https://expressjs.com/)
