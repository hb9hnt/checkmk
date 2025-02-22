<!--
Copyright (C) 2024 Checkmk GmbH - License: GNU General Public License v2
This file is part of Checkmk (https://checkmk.com). It is subject to the terms and
conditions defined in the file COPYING, which is part of this source code package.
-->
<script lang="ts">
import { h, type VNode, defineComponent, type PropType } from 'vue'
import type {
  Components,
  Dictionary,
  FormSpec,
  List,
  TimeSpan,
  SingleChoice,
  SingleChoiceElement,
  CascadingSingleChoice,
  LegacyValuespec,
  FixedValue,
  BooleanChoice,
  MultilineText,
  Password,
  Tuple,
  OptionalChoice,
  ListOfStrings,
  DualListChoice,
  CheckboxListChoice,
  Folder,
  Labels,
  ConditionChoices,
  ConditionChoicesValue,
  ConditionGroup,
  TimeSpecific,
  FileUpload
} from 'cmk-shared-typing/typescript/vue_formspec_components'
import {
  groupNestedValidations,
  groupIndexedValidations,
  type ValidationMessages
} from '@/form/components/utils/validation'
import { splitToUnits, getSelectedMagnitudes, ALL_MAGNITUDES } from './utils/timeSpan'
import {
  translateOperator,
  type Operator,
  type OperatorI18n
} from './forms/FormConditionChoices/utils'

function renderForm(
  formSpec: FormSpec,
  value: unknown,
  backendValidation: ValidationMessages = []
): VNode {
  switch (formSpec.type as Components['type']) {
    case 'dictionary':
      return renderDict(formSpec as Dictionary, value as Record<string, unknown>, backendValidation)
    case 'time_span':
      return renderTimeSpan(formSpec as TimeSpan, value as number)
    case 'string':
    case 'integer':
    case 'float':
    case 'metric':
      return renderSimpleValue(formSpec, value as string, backendValidation)
    case 'single_choice_editable':
    case 'single_choice':
      return renderSingleChoice(formSpec as SingleChoice, value as unknown, backendValidation)
    case 'list':
    case 'list_unique_selection':
      return renderList(formSpec as List, value as unknown[], backendValidation)
    case 'list_of_strings':
      return renderListOfStrings(formSpec as ListOfStrings, value as unknown[], backendValidation)
    case 'cascading_single_choice':
      return renderCascadingSingleChoice(
        formSpec as CascadingSingleChoice,
        value as [string, unknown],
        backendValidation
      )
    case 'condition_choices':
      return renderConditionChoices(formSpec as ConditionChoices, value as ConditionChoicesValue[])
    case 'legacy_valuespec':
      return renderLegacyValuespec(formSpec as LegacyValuespec, value, backendValidation)
    case 'fixed_value':
      return renderFixedValue(formSpec as FixedValue)
    case 'boolean_choice':
      return renderBooleanChoice(formSpec as BooleanChoice, value as boolean)
    case 'multiline_text':
    case 'comment_text_area':
      return renderMultilineText(formSpec as MultilineText, value as string)
    case 'data_size':
      return renderDataSize(value as [string, string])
    case 'catalog':
      return h('div', 'Catalog does not support readonly')
    case 'dual_list_choice':
      return renderMultipleChoice(formSpec as DualListChoice, value as string[])
    case 'checkbox_list_choice':
      return renderMultipleChoice(formSpec as CheckboxListChoice, value as string[])
    case 'password':
      return renderPassword(formSpec as Password, value as (string | boolean)[])
    case 'tuple':
      return renderTuple(formSpec as Tuple, value as unknown[])
    case 'optional_choice':
      return renderOptionalChoice(formSpec as OptionalChoice, value as unknown[])
    case 'simple_password':
      return renderSimplePassword()
    case 'folder':
      return renderFolder(formSpec as Folder, value as string, backendValidation)
    case 'labels':
      return renderLabels(formSpec as Labels, value as Record<string, string>)
    case 'time_specific':
      return renderTimeSpecific(formSpec as TimeSpecific, value, backendValidation)
    case 'file_upload':
      return renderFileUpload(formSpec as FileUpload, value as [string, string, string])
    // Do not add a default case here. This is intentional to make sure that all form types are covered.
  }
}

function renderSimplePassword(): VNode {
  return h('div', ['******'])
}

function renderFileUpload(_formSpec: FileUpload, value: [string, string, string]): VNode {
  return h('div', [value[0]])
}

function renderOptionalChoice(
  formSpec: OptionalChoice,
  value: unknown,
  backendValidation: ValidationMessages = []
): VNode {
  if (value === null) {
    return h('div', formSpec.i18n.none_label)
  }
  const embeddedMessages: ValidationMessages = []
  const localMessages: ValidationMessages = []
  backendValidation.forEach((msg) => {
    if (msg['location'].length > 0) {
      embeddedMessages.push({
        location: msg.location.slice(1),
        message: msg.message,
        invalid_value: msg.invalid_value
      })
    } else {
      localMessages.push(msg)
    }
  })

  const errorFields: VNode[] = []
  localMessages.forEach((msg) => {
    errorFields.push(h('label', [msg.message]))
  })

  const embeddedResult = renderForm(formSpec.parameter_form, value, embeddedMessages)
  return h('div', [errorFields.concat(embeddedResult ? [embeddedResult] : [])])
}

function renderTuple(
  formSpec: Tuple,
  value: unknown[],
  backendValidation: ValidationMessages = []
): VNode {
  const [tupleValidations, elementValidations] = groupIndexedValidations(
    backendValidation,
    value.length
  )
  return h(
    'div',
    { class: `form-readonly__tuple form-readonly__tuple__layout-${formSpec.layout}` },
    [
      ...tupleValidations.map((validation: string) => {
        return h('label', [validation])
      }),
      ...formSpec.elements.map((element, index) => {
        // @ts-expect-error label does not exist on all element
        const title: string = element.title || element['label']
        return h('span', [
          formSpec.show_titles && title ? `${title}: ` : h([]),
          renderForm(element, value[index], elementValidations[index]),
          index === formSpec.elements.length - 1 ? '' : ', '
        ])
      })
    ]
  )
}

function renderMultipleChoice(
  formSpec: DualListChoice | CheckboxListChoice,
  value: string[]
): VNode {
  const nameToTitle: Record<string, string> = {}
  for (const element of formSpec.elements) {
    nameToTitle[element.name] = element.title
  }

  const maxEntries = 5
  const textSpans: VNode[] = []

  // WIP: no i18n...
  for (const [index, entry] of value.entries()) {
    if (index >= maxEntries) {
      break
    }
    textSpans.push(h('span', nameToTitle[entry]!))
  }
  if (value.length > maxEntries) {
    textSpans.push(
      h(
        'span',
        { class: 'form-readonly__multiple-choice__max-entries' },
        ` and ${value.length - maxEntries} more`
      )
    )
  }
  return h('div', { class: 'form-readonly__multiple-choice' }, textSpans)
}

function renderDataSize(value: [string, string]): VNode {
  return h('div', [h('span', value[0]), h('span', ' '), h('span', value[1])])
}

function renderMultilineText(formSpec: MultilineText, value: string): VNode {
  const lines: VNode[] = []
  value.split('\n').forEach((line) => {
    lines.push(h('span', { style: 'white-space: pre-wrap' }, line))
    lines.push(h('br'))
  })

  const style = formSpec.monospaced ? 'font-family: monospace, sans-serif' : ''
  return h('div', { style: style }, lines)
}

function renderBooleanChoice(formSpec: BooleanChoice, value: boolean): VNode {
  return h('div', value ? formSpec.text_on : formSpec.text_off)
}

function renderFixedValue(formSpec: FixedValue): VNode {
  let shownValue = formSpec.value
  if (formSpec.label !== null) {
    shownValue = formSpec.label
  }
  return h('div', shownValue as string)
}

function renderDict(
  formSpec: Dictionary,
  value: Record<string, unknown>,
  backendValidation: ValidationMessages
): VNode {
  const dictElements: VNode[] = []
  // Note: Dictionary validations are not shown
  const [, elementValidations] = groupNestedValidations(formSpec.elements, backendValidation)
  formSpec.elements.map((element) => {
    if (value[element.name] === undefined) {
      return
    }

    const elementForm = renderForm(
      element.parameter_form,
      value[element.name],
      elementValidations[element.name] || []
    )
    if (elementForm === null) {
      return
    }
    dictElements.push(
      h('tr', [h('th', `${element.parameter_form.title}: `), h('td', [elementForm])])
    )
  })
  const cssClasses = [
    'form-readonly__dictionary',
    formSpec.layout === 'two_columns' ? 'form-readonly__dictionary--two_columns' : ''
  ]
  return h('table', { class: cssClasses }, dictElements)
}

function computeUsedValue(
  value: unknown,
  backendValidation: ValidationMessages = []
): [string, boolean, string] {
  if (backendValidation.length > 0) {
    return [backendValidation[0]!.invalid_value as string, true, backendValidation[0]!.message]
  }
  return [value as string, false, '']
}

function renderSimpleValue(
  _formSpec: FormSpec,
  value: string,
  backendValidation: ValidationMessages = []
): VNode {
  const [usedValue, isError, errorMessage] = computeUsedValue(value, backendValidation)
  const cssClasses = ['form-readonly__simple-value', isError ? 'form-readonly__error' : '']
  return h('div', { class: cssClasses }, isError ? [`${usedValue} - ${errorMessage}`] : [usedValue])
}

function renderPassword(formSpec: Password, value: (string | boolean)[]): VNode {
  if (value[0] === 'explicit_password') {
    return h('div', [`${formSpec.i18n.explicit_password}, ******`])
  }

  const storeChoice = formSpec.password_store_choices.find(
    (choice) => choice.password_id === value[1]
  )
  if (storeChoice) {
    return h('div', [`${formSpec.i18n.password_store}, ${storeChoice.name}`])
  } else {
    return h('div', [`${formSpec.i18n.password_store}, ${formSpec.i18n.password_choice_invalid}`])
  }
}

function renderTimeSpan(formSpec: TimeSpan, value: number): VNode {
  const result = []
  const values = splitToUnits(value, getSelectedMagnitudes(formSpec.displayed_magnitudes))
  for (const [magnitude] of ALL_MAGNITUDES) {
    const v = values[magnitude]
    if (v !== undefined) {
      result.push(`${v} ${formSpec.i18n[magnitude]}`)
    }
  }
  return h('div', [result.join(' ')])
}

function renderSingleChoice(
  formSpec: { elements: SingleChoiceElement[] },
  value: unknown,
  backendValidation: ValidationMessages = []
): VNode {
  for (const element of formSpec.elements) {
    if (element.name === value) {
      return h('div', [element.title])
    }
  }

  // Value not found in valid values. Try to show error
  const [usedValue, isError, errorMessage] = computeUsedValue(value, backendValidation)
  if (isError) {
    return h('div', { class: 'form-readonly__error' }, [`${usedValue} - ${errorMessage}`])
  }

  // In case no validation message is present, we still want to show raw_value
  // (This should not happen in production, but is useful for debugging)
  return h('div', { class: 'form-readonly__error' }, [`${usedValue} - Invalid value`])
}

function renderList(
  formSpec: List,
  value: unknown[],
  backendValidation: ValidationMessages
): VNode {
  const [listValidations, elementValidations] = groupIndexedValidations(
    backendValidation,
    value.length
  )
  if (!value) {
    return h([])
  }
  const listResults = [h('label', [formSpec.element_template.title])]
  listValidations.forEach((validation: string) => {
    listResults.push(h('label', [validation]))
  })

  for (let i = 0; i < value.length; i++) {
    listResults.push(
      h('li', [
        renderForm(
          formSpec.element_template,
          value[i],
          elementValidations[i] ? elementValidations[i] : []
        )
      ])
    )
  }
  return h('ul', { class: 'form-readonly__list' }, listResults)
}

function renderListOfStrings(
  formSpec: ListOfStrings,
  value: unknown[],
  backendValidation: ValidationMessages
): VNode {
  const [listValidations, elementValidations] = groupIndexedValidations(
    backendValidation,
    value.length
  )
  if (!value) {
    return h([])
  }
  const listResults = [h('label', [formSpec.string_spec.title])]
  listValidations.forEach((validation: string) => {
    listResults.push(h('label', [validation]))
  })

  for (let i = 0; i < value.length; i++) {
    listResults.push(
      h('li', [
        renderForm(
          formSpec.string_spec,
          value[i],
          elementValidations[i] ? elementValidations[i] : []
        )
      ])
    )
  }
  return h('ul', { style: 'display: contents' }, listResults)
}

function renderCascadingSingleChoice(
  formSpec: CascadingSingleChoice,
  value: [string, unknown],
  backendValidation: ValidationMessages
): VNode {
  for (const element of formSpec.elements) {
    if (element.name === value[0]) {
      return h(
        'div',
        h('div', { class: `form-readonly__cascading-single-choice__layout-${formSpec.layout}` }, [
          // notification explicitly defines empty string title:
          [formSpec.title ? h('div', formSpec.title) : []],
          h('div', element.title),
          h('div', [renderForm(element.parameter_form, value[1], backendValidation)])
        ])
      )
    }
  }
  return h([])
}

function renderTimeSpecific(
  formSpec: TimeSpecific,
  value: unknown,
  backendValidation: ValidationMessages
): VNode {
  const isActive = typeof value === 'object' && value !== null && 'tp_default_value' in value
  if (isActive) {
    return h('div', [renderForm(formSpec.parameter_form_enabled, value, backendValidation)])
  } else {
    return h('div', [renderForm(formSpec.parameter_form_disabled, value, backendValidation)])
  }
}

interface PreRenderedHtml {
  input_html: string
  readonly_html: string
}

function renderLegacyValuespec(
  _formSpec: LegacyValuespec,
  value: unknown,
  backendValidation: ValidationMessages
): VNode {
  return h('div', [
    h('div', {
      style: 'background: #595959',
      class: 'legacy_valuespec',
      innerHTML: (value as PreRenderedHtml).readonly_html
    }),
    h('div', { validation: backendValidation })
  ])
}

function renderConditionChoiceGroup(
  group: ConditionGroup,
  value: ConditionChoicesValue,
  i18n: OperatorI18n
): VNode {
  const condition = Object.values(value.value)[0] as string | string[]
  const conditionList = condition instanceof Array ? condition : [condition]
  const operator = translateOperator(i18n, Object.keys(value.value)[0] as Operator)
  return h('tr', [
    h('td', group.title),
    h('td', [
      h('table', [
        h('tbody', [
          h('th', h('b', operator)),
          ...conditionList.map((cValue) =>
            h('tr', h('td', group.conditions.find((cGroup) => cGroup.name === cValue)?.title))
          )
        ])
      ])
    ])
  ])
}

function renderConditionChoices(formSpec: ConditionChoices, value: ConditionChoicesValue[]): VNode {
  return h('table', { class: 'form-readonly__table' }, [
    h('tbody', [
      value.map((v) => {
        const group = formSpec.condition_groups[v.group_name]
        if (group === undefined) {
          throw new Error('Invalid group')
        }
        return renderConditionChoiceGroup(group, v, formSpec.i18n)
      })
    ])
  ])
}

function renderFolder(
  formSpec: Folder,
  value: string,
  backendValidation: ValidationMessages
): VNode {
  return renderSimpleValue(formSpec, `Main/${value}`, backendValidation)
}
function renderLabels(formSpec: Labels, value: Record<string, string>): VNode {
  let bgColor = 'var(--tag-color)'
  let color = 'var(--black)'

  switch (formSpec.label_source) {
    case 'discovered':
      bgColor = 'var(--tag-discovered-color)'
      color = 'var(--white)'
      break
    case 'explicit':
      bgColor = 'var(--tag-explicit-color)'
      color = 'var(--black)'
      break
    case 'ruleset':
      bgColor = 'var(--tag-ruleset-color)'
      color = 'var(--white)'
      break
  }
  return h(
    'div',
    { class: 'form-readonly__labels' },
    Object.entries(value).map(([key, value]) => {
      return h(
        'div',
        { class: 'label', style: { backgroundColor: bgColor, color: color } },
        `${key}: ${value}`
      )
    })
  )
}

export default defineComponent({
  props: {
    spec: { type: Object as PropType<FormSpec>, required: true },
    data: { type: null as unknown as PropType<unknown>, required: true },
    backendValidation: { type: Object as PropType<ValidationMessages>, required: true }
  },
  render() {
    return renderForm(this.spec, this.data, this.backendValidation)
  }
})
</script>

<style scoped>
.form-readonly__error {
  background-color: var(--form-readonly-error-bg-color);
}

table.form-readonly__table {
  margin-top: 3px;
  border-collapse: collapse;

  td,
  th {
    background: var(--default-table-th-color);
  }

  td {
    height: 14px;
    padding: 1px 5px;
    border: 1px solid grey;
  }

  table {
    border-collapse: collapse;
    padding: 1px 5px;
  }
}

.form-readonly__simple-value {
  display: inline-block;
}

.form-readonly__dictionary {
  display: inline-table;

  > tr > th {
    padding-right: var(--spacing-half);
  }
}

.form-readonly__dictionary--two_columns > tr {
  line-height: 18px;

  > th {
    width: 130px;
    padding-right: 16px;
    font-weight: var(--font-weight-bold);
  }

  > td > .form-readonly__multiple-choice span {
    display: block;

    &:before {
      content: '';
    }
  }
}

.form-readonly__multiple-choice span {
  &:not(:first-child):before {
    content: ', ';
  }

  &.form-readonly__multiple-choice__max-entries:before {
    content: '';
  }
}

.form-readonly__list {
  padding-left: 0;
  list-style-position: inside;
}

.form-readonly__list > li > div {
  display: inline-block;
}

.form-readonly__labels {
  display: flex;
  flex-direction: row;
  justify-content: start;
  align-items: center;
  flex-wrap: wrap;
  gap: 5px 0;

  > .label {
    width: fit-content;
    margin: 0 5px 0 0;
    padding: 1px 4px;
    border-radius: 5px;

    &:first-child {
      margin-left: 0;
    }
  }
}

.form-readonly__cascading-single-choice__layout-horizontal {
  display: flex;
  flex-direction: column;
  margin-bottom: 4px;
}
.form-readonly__cascading-single-choice__layout-horizontal > div {
  margin-right: var(--spacing-half);
}

.form-readonly__tuple > span > * {
  display: inline;
}

.form-readonly__tuple__layout-vertical > span {
  display: block;
}
</style>
