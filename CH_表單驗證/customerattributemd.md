# 自訂資料驗證屬性
當Asp.Net內建的資料驗證屬性無法滿足需求時, 需要自行建立客製化的驗證屬性.

找到一篇[ASP.NET MVC CheckBox Validations (server and client side)](http://devfarm.it/asp-net-mvc-2/asp-net-mvc-checkbox-validations-server-client-side/)的範例, 按照它的方法來做可以擴充對表單的Checkbox做前後端的驗證.

### Problem
你希望可以驗證表單中Checkbox在沒有勾選的情況下, 不管是做送出或沒點選, 會提示使用者錯誤訊息.

### Solution
1. 建立類別, 並繼承```ValidationAttribute, IClientValidatable```, 程式碼如下:
```
    [AttributeUsage(AttributeTargets.Property, AllowMultiple = false, Inherited = false)]
    public class MustBeTrueAttribute : ValidationAttribute, IClientValidatable
    {
        protected override ValidationResult IsValid(object value, ValidationContext validationContext)
        {
            if ((bool)value)
                return ValidationResult.Success;
            return new ValidationResult(String.Format(ErrorMessageString, validationContext.DisplayName));
        }

        public IEnumerable<ModelClientValidationRule> GetClientValidationRules(ModelMetadata metadata, ControllerContext context)
        {
            var rule = new ModelClientValidationRule
            {
                ErrorMessage = FormatErrorMessage(metadata.GetDisplayName()),
                ValidationType = "checkrequired"
            };

            yield return rule;
        }
    }
```
2. 前端驗證的部份需擴充jQuery validation
這裡要注意的是擴充的程式必需要在jQuery相關的函式庫都載入完成後才可以執行這段程式.
```
if (jQuery.validator) {
    // Checkbox Validation
    jQuery.validator.addMethod("checkrequired", function (value, element, params) {
        var checked = false;
        checked = $(element).is(':checked');
        return checked;
    }, '');
    if (jQuery.validator.unobtrusive) {
        jQuery.validator.unobtrusive.adapters.addBool("checkrequired");
    }
}
```
3. 在對應表單的類別中, 使用自訂驗證屬性
```
    public class FormContact
    {

        [Display(Name = "Name")]
        [Required]
        public string Name { get; set; }

        [Display(Name = "Email")]
        [DataType(DataType.EmailAddress)]
        [Required]
        public string Email { get; set; }

        [Display(Name = "Privacy")]
        [MustBeTrueAttribute(ErrorMessage = "Please Accept Privacy Policy")]
        public bool Privacy { get; set; }

    }
```

### Discussion
Asp.Net提供了很好的擴充性來自訂驗證屬性, 而它能做的不只是對bool型別, 對於string型別也是可以自行擴充, 並可傳入要比對的特定字串, 範例中, 只要輸入mvc字串就會驗證失敗:
```
//在客製化的attribute類別中:
//範例需求: 需要有一個傳入值,以判斷傳入值是否與我們要比較的字串相等.
public NoIsAttribute(string input)
{
    Input = input;
}

public override bool IsValid(object value)
{
    if (value.Equals(null))
        return true;
    //輸入值與欄位值相同就報錯
    if (value is string)
        return !Input.Equals(value.ToString());

    return true;
}

//表單類別
public class CustomTestModels
{
    [NoIs("mvc")]
    public string Name { get; set; }
}
```
前端的Javascript:
```
@section Scripts {
    @Scripts.Render("~/bundles/jqueryval")

    <script type="text/javascript">
            $.validator.addMethod("nois", function (value, element, param) {
                if (value === false) {
                    return true;
                }
                if (value.indexOf(param) != -1) {
                    return false;
                }
                else {
                    return true;
                }
            });
            $.validator.unobtrusive.adapters.addSingleVal("nois", "input");
    </script>
}
```