---
#date: '2026-02-16T10:52:19+01:00'

tags: ["gameproject"]
#image: http://localhost:1313/images/profile.png
image: images/TSRS_Thumbnail.png
featured: true
title: 'Game Project: Tiny Shell Rough Sea'

---
<!-- ![white kitten](/images/TSRS_Thumbnail.png "A kitten!") -->

My Contributions:

- **UI**
- **Audio**

This was my first game project at The Game Assembly, made in 5 weeks in Unity (Version 2022.3). Our team made a simple Race Against the Sun type game where you play as a sea turtle and dodge obstacles in your way.

## Buttons 

![](/images/TSRS_HoverButtons.gif "A kitten!")

On this project my main task was working with the game's UI. Setting up the base functionality for the menus is in unity is straight forward, since Unity already has working buttons built in that can be connected to any C# script function. Later into the project we decided we wanted our buttons to have hover effects and since Unity's buttons did not supprot that I had to implement my own button script.

Making a button script in Unity is still straight forward, since you can expose a UnityEvent variable in the script to connect any C# script function. Unity also has Interfaces for pointer events, so using those made the script simple.

```cs 
public class Script_AdvancedButton : MonoBehaviour, IPointerDownHandler,IPointerEnterHandler,IPointerExitHandler
{
    [SerializeField] UnityEvent OnClick;

    [Header("Size change when hovering")]
    [SerializeField] float myHoverScaleModifier;
    [SerializeField] float myScaleOvershoot;
    [SerializeField] float myOvershootFalloff = 4;

    [Header("Rotation")]
    [SerializeField] float rotationsSpeed;

    [Header("Colors")]
    [SerializeField] Color myDefaultColor = Color.white;
    [SerializeField] Color myDisabledColor = Color.white;

    [SerializeField] Image myImage;

    float myCurrentScaleOvershoot;
    float myScaleDirection;

    bool myIsHovered;
    

    public bool Interactable 
    { 
        get
        {
            return myInteractable;
        }
        
        set
        {
            myInteractable = value;
            myImage.color = myInteractable ? myDefaultColor : myDisabledColor;
        }
    }
    [SerializeField] bool myInteractable = true;

    private void Awake()
    {
        if(myImage == null) myImage = GetComponent<Image>();
    }

    private void Update()
    {
        Vector3 targetSize = Vector3.one;
        if(myIsHovered)
        {
            transform.rotation *= Quaternion.Euler(0f, 0f, rotationsSpeed * Time.unscaledDeltaTime);
            targetSize *= myHoverScaleModifier + myCurrentScaleOvershoot * myScaleDirection;

            if (myCurrentScaleOvershoot > 0.01f)
            {
                if(transform.localScale == targetSize)
                {
                    myCurrentScaleOvershoot /= myOvershootFalloff;
                    myScaleDirection *= -1f;
                }
            }
        }

        transform.localScale = Vector3.MoveTowards(transform.localScale, targetSize, Time.unscaledDeltaTime * 10f);
        
    }

    private void OnDisable()
    {
        myIsHovered = false;
        transform.localScale = Vector3.one;
    }

    public void OnPointerDown(PointerEventData eventData)
    {
        if (!Interactable)
        {
            return;
        }
        Script_SfxManager.PlaySound(Script_SfxManager.Sound.Click);
        OnClick?.Invoke();
    }

    public void OnPointerEnter(PointerEventData eventData)
    {
        if(!Interactable)
        {
            return;
        }

        Script_SfxManager.PlaySound(Script_SfxManager.Sound.Hover);

        myIsHovered = true;
        myCurrentScaleOvershoot = myScaleOvershoot;
        myScaleDirection = 1f;
    }

    public void OnPointerExit(PointerEventData eventData)
    {
        if (!Interactable)
        {
            return;
        }
        myIsHovered = false;
    }
}

```

The first effect i made was making the buttons expand when hovered. This was done by setting a target scale variable, depending on if the button is hovered or not, and smoothly moving the local scale to the target value. 


Since our button sprites would be bubbles, the hover effect was then made to be bouncy. I did this by alternating between a greater and smaller target scale, with a overshoot variable. That variable would then decrease every bounce to smoothly stop the bouncing

## GUI

In our game we have a mechanic where you gain a greater score multiplier the less oxygen you had. Something we noticed turing playtests was that most people did not understand that mechanic. To make it easier to understand, I added a bounce effect to the multipliers once they became active.

![](/images/TSRS_BounceText.gif "A kitten!")

<!-- <img src="/images/TSRS_Thumbnail.png" alt="My Example Image" width="600" height="400"> -->

## Audio

Since this was a vert short project, the sound effects script remained simple throughout. The script contained an array with all the audio clips, and a coresponding enum value for every sound. The script was then made to be a singleton, allowing calls to the PlaySound funciton from anywere and therefore making it easy to implement sound effects.

For some sound effects that could not be played every frame, like slider value change sound, a separate function was made to pervent the sound from being played again until a duration has passed.

```cs
public class Script_SfxManager : MonoBehaviour
{
    static Script_SfxManager ourInstance;
    static AudioSource ourSource;

    [SerializeField] AudioClip[] clips;

    static readonly List<Sound> waitingSounds = new();
    static readonly List<float> waitingTimers = new();

    public enum Sound
    {
        Click,
        SuperClick,
        Hover,
        Pause,
        Hover2,
        Drown,
        Crash,
        QuickTurn,
        JellyFish,
        Bubble,
        SideCollision,
        Transition,
    }

    private void Awake()
    {
        if(ourInstance == null) 
        {
            ourInstance = this;
            ourSource = GetComponent<AudioSource>();
            DontDestroyOnLoad(gameObject);
        }
        else
        {
            Destroy(gameObject);
        }
    }

    private void Update()
    {
        float deltaTime = Time.unscaledDeltaTime;
        for(int i = 0; i < waitingTimers.Count; i++)
        {
            waitingTimers[i] -= deltaTime;
            if (waitingTimers[i] < 0 )
            {
                waitingTimers.RemoveAt(i);
                waitingSounds.RemoveAt(i);
                i--;
            }
        }
    }

    /// <summary>
    /// Plays a sound
    /// </summary>
    public static void PlaySound(Sound aSound,float aVolume = 0.75f)
    {
        ourSource.PlayOneShot(ourInstance.clips[(int)aSound], aVolume);
    }

    /// <summary>
    /// Plays a sound, and pervents that sound from being played for the aWaitDuration (in seconds)
    /// </summary>
    public static void PlaySoundLimit(Sound aSound,float aWaitDuration)
    {
        if(waitingSounds.Contains(aSound))
        {
            return;
        }

        ourSource.PlayOneShot(ourInstance.clips[(int)aSound]);
        waitingSounds.Add(aSound);
        waitingTimers.Add(aWaitDuration);
    }
}

```