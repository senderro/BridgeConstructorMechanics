# BridgeConstructorMechanic 2D
## Hello everyone
In this repository are some codes that I used to create a 2D bridge builder, the code was not created from scratch by me I saw in a tutorial.


```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class Bar : MonoBehaviour
{
    public float costForLength = 1;
    public float totalCost;
    public float maxLength = 1f;
    public Vector2 startPosition;
    public SpriteRenderer barSpriterRenderer;

    public BoxCollider2D boxCollider2D;
    public HingeJoint2D StartHingeJoint;
    public HingeJoint2D EndHingeJoint;

    float startJointCurrentLoad = 0;
    float endJointCurrentLoad = 0;

    MaterialPropertyBlock materialPropertyBlock;
    public void UpdateCreatingBar(Vector2 ToPosition){
        transform.position = (startPosition + ToPosition)/2;

        Vector2 direction = ToPosition - startPosition;
        float angle = Vector2.SignedAngle(Vector2.right, direction);
        transform.rotation = Quaternion.Euler(new Vector3(0,0,angle-90));

        float length = direction.magnitude;
        barSpriterRenderer.size = new Vector2(barSpriterRenderer.size.x, length);

        boxCollider2D.size = barSpriterRenderer.size;

        totalCost = length * costForLength;

    }

    public void UpdateMaterial()
    {
        if ( StartHingeJoint != null) startJointCurrentLoad = StartHingeJoint.reactionForce.magnitude / StartHingeJoint.breakForce;
        if ( EndHingeJoint != null) endJointCurrentLoad = EndHingeJoint.reactionForce.magnitude / EndHingeJoint.breakForce;
        float maxLoad = Mathf.Max(startJointCurrentLoad, endJointCurrentLoad);

        materialPropertyBlock = new MaterialPropertyBlock();
        barSpriterRenderer.GetPropertyBlock(materialPropertyBlock);
        materialPropertyBlock.SetFloat("_Load", maxLoad);
        barSpriterRenderer.SetPropertyBlock(materialPropertyBlock);
    }

    private void Update(){
        if (Time.timeScale == 1)
        {
            UpdateMaterial();
        }
    }
}


```

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

[ExecuteInEditMode]
public class Point : MonoBehaviour
{
    public bool Runtime = true;
    public List<Bar> ConnectedBars;

    public Vector2 PointID;

    public Rigidbody2D rbd;


    private void Start(){
        if(Runtime == false){
            PointID = transform.position;
            rbd.bodyType = RigidbodyType2D.Static;
            if(GameManager.AllPoints.ContainsKey(PointID) == false){
                GameManager.AllPoints.Add(PointID, this);
            }
        }
    }
    private void Update(){
        if(Runtime == false){
            if(transform.hasChanged == true){
                transform.hasChanged = false;
                transform.position = Vector3Int.RoundToInt(transform.position);
            }
        }
    }
}

```

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.EventSystems;

public class BarCreator : MonoBehaviour, IPointerDownHandler
{
    public GameManager MyGameManager;
    public GameObject RoadBar;
    public GameObject WoodBar;
    bool barCreationStarted = false;
    public Bar CurrentBar;
    public GameObject BarPrefab;
    public Transform BarParent;

    public Point CurrentStartPoint;
    public Point CurrentEndPoint;
    public GameObject PointPrefab;
    public Transform PointParent;
    public void OnPointerDown(PointerEventData eventData)
    {
        if(barCreationStarted == false){
            barCreationStarted = true;
            StartBarCreation(Vector2Int.RoundToInt(Camera.main.ScreenToWorldPoint(eventData.position)));
        }else{
            if(eventData.button == PointerEventData.InputButton.Left)
            {
                if(MyGameManager.SpendDin(CurrentBar.totalCost) == true)
                {
                FinishBarCreation();
                }
            }
            else if(eventData.button == PointerEventData.InputButton.Right)
            {
                barCreationStarted = false;
                CancelBarCreation();
            }
        }
    }

    void StartBarCreation(Vector2 StartPosition)
    {
        CurrentBar = Instantiate(BarPrefab, BarParent).GetComponent<Bar>();
        CurrentBar.startPosition = StartPosition;

        if(GameManager.AllPoints.ContainsKey(StartPosition)){
            CurrentStartPoint = GameManager.AllPoints[StartPosition];
        }else{
            CurrentStartPoint = Instantiate(PointPrefab, StartPosition, Quaternion.identity, PointParent).GetComponent<Point>();
            GameManager.AllPoints.Add(StartPosition, CurrentStartPoint);
        }
        CurrentEndPoint = Instantiate(PointPrefab, StartPosition, Quaternion.identity, PointParent).GetComponent<Point>();
    }

    void FinishBarCreation(){
        if(GameManager.AllPoints.ContainsKey(CurrentEndPoint.transform.position)){
            Destroy(CurrentEndPoint.gameObject);
            CurrentEndPoint = GameManager.AllPoints[CurrentEndPoint.transform.position];
        }else{
            GameManager.AllPoints.Add(CurrentEndPoint.transform.position, CurrentEndPoint);
        }
        CurrentStartPoint.ConnectedBars.Add(CurrentBar);
        CurrentEndPoint.ConnectedBars.Add(CurrentBar);

        CurrentBar.StartHingeJoint.connectedBody = CurrentStartPoint.rbd;
        CurrentBar.StartHingeJoint.anchor = CurrentBar.transform.InverseTransformPoint(CurrentBar.startPosition);

        CurrentBar.EndHingeJoint.connectedBody = CurrentEndPoint.rbd;
        CurrentBar.EndHingeJoint.anchor = CurrentBar.transform.InverseTransformPoint(CurrentEndPoint.transform.position);

        MyGameManager.UpdateDin(CurrentBar.totalCost);

        StartBarCreation(CurrentEndPoint.transform.position);
    }

    private void Update(){
        if(barCreationStarted == true){
            Vector2 endposition = (Vector2)Vector2Int.RoundToInt(Camera.main.ScreenToWorldPoint(Input.mousePosition));
            Vector2 direction = endposition - CurrentBar.startPosition;
            Vector2 raioPosicao = CurrentBar.startPosition + Vector2.ClampMagnitude(direction, CurrentBar.maxLength);

            CurrentEndPoint.transform.position = (Vector2)Vector2Int.FloorToInt(raioPosicao);
            CurrentEndPoint.PointID = CurrentEndPoint.transform.position;
            CurrentBar.UpdateCreatingBar(CurrentEndPoint.transform.position);
        }
    }

    void CancelBarCreation()
    {
        Destroy(CurrentBar.gameObject);
        if(CurrentStartPoint.ConnectedBars.Count==0 && CurrentStartPoint.Runtime==true) Destroy(CurrentStartPoint.gameObject);
        if(CurrentEndPoint.ConnectedBars.Count==0 && CurrentEndPoint.Runtime==true) Destroy(CurrentEndPoint.gameObject);
    }
}

```

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class GameManager : MonoBehaviour
{
    public static Dictionary<Vector2, Point> AllPoints = new Dictionary<Vector2, Point>(); 
    public float levelDin = 1000;
    public float currentDin = 0;
    public UIManager myUIManager; 
    private void Awake()
    {
        AllPoints.Clear();
        Time.timeScale = 0;
        currentDin = levelDin;
        myUIManager.UpdateSlider(currentDin, levelDin);
    }

    public bool SpendDin(float amount)
    {
        if(currentDin - amount >= 0)
        {
            return true;
        }
        return false;
    }

    public void UpdateDin(float amount)
    {
        currentDin -= amount;
        myUIManager.UpdateSlider(currentDin, levelDin);
    }

    [ContextMenu("TestePlay")]
    public void Play()
    {
        Time.timeScale = 1;
    }
}

```

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;
using UnityEngine.SceneManagement;
using TMPro;
public class UIManager : MonoBehaviour
{
    public Button RoadButton;
    public Button WoodButton;
    public BarCreator barCreator;

    public Slider slider;
    public TMP_Text sliderText;
    public Gradient gradient;

    private void Start(){
        RoadButton.onClick.Invoke();
    }

    public void Play()
    {
        Time.timeScale = 1;
    }

    public void Restart(){
        SceneManager.LoadScene("SampleScene");
    }

    public void ChangeBar (int barType){
        if(barType == 0 ){
            WoodButton.GetComponent<Outline>().enabled = false; 
            RoadButton.GetComponent<Outline>().enabled = true; 
            barCreator.BarPrefab = barCreator.RoadBar;
        }else if(barType == 1){
            WoodButton.GetComponent<Outline>().enabled = true; 
            RoadButton.GetComponent<Outline>().enabled = false; 
            barCreator.BarPrefab = barCreator.WoodBar;
        }
    }

    public void UpdateSlider(float currentDin, float levelDin){
        sliderText.text = Mathf.FloorToInt(currentDin).ToString("") + "$";
        slider.value = currentDin/levelDin;
        slider.fillRect.GetComponent<Image>().color = gradient.Evaluate(slider.value);
    }
}

```

