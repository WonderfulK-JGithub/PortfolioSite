---
#date: '2026-02-16T10:52:19+01:00'
date: '2024-02-16T10:52:19+01:00'
tags: ["gameproject","ui","unity"]
image: images/TSRS_Thumbnail2.png
title: 'Tiny Shell Rough Sea'
subtitle: "a desktop game made in Unity in 6 weeks with a team of 15"
github: https://github.com/WonderfulK-JGithub/Tiny-Shell-Rough-Sea
hideThumbnail: true
---
{{< youtube id=0KOZCcSho6c   >}}

🎮 Link to game: [https://coahl.itch.io/tiny-shell-rough-sea](https://coahl.itch.io/tiny-shell-rough-sea)

My Contributions:

- **UI**
- **Audio**
- **Scene Transitions**

This was my first game project at The Game Assembly, made in 5 weeks in Unity (Version 2022.3). Our team made a simple Race Against the Sun type game where you play as a sea turtle and dodge obstacles in your way.

## Buttons 

![](../../images/TSRS_HoverButtons.gif "hover buttons!")

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

The first effect I made was making the buttons expand when hovered. This was done by setting a target scale variable, depending on if the button is hovered or not, and smoothly moving the local scale to the target value. 


Since our button sprites would be bubbles, the hover effect was then made to be bouncy. I did this by alternating between a greater and smaller target scale, with a overshoot variable. That variable would then decrease every bounce to smoothly stop the bouncing

## GUI

In our game we have a mechanic where you gain a greater score multiplier the less oxygen you had. Something we noticed turing playtests was that most people did not understand that mechanic. To make it easier to understand, I added a bounce effect to the multipliers once they became active.

![](../../images/TSRS_BounceText.gif "bounce text!")